# Rails에서 `telegram-bot-ruby`로 Telegram Webhook 붙이기

이 gem은 기본적으로 long polling 예제를 중심으로 설명하지만, webhook에 필요한 핵심 조각은 이미 제공합니다.

- Telegram Bot API 호출: `set_webhook`, `delete_webhook`, `get_webhook_info`
- typed update 파싱: `Telegram::Bot::Types::Update.new`
- update에서 현재 이벤트 꺼내기: `update.current_message`

반대로, Telegram이 호출할 inbound HTTP 엔드포인트는 이 gem이 직접 제공하지 않습니다. webhook을 쓰려면 Rails 앱에서 route, controller, initializer, 그리고 봇 로직 클래스를 직접 연결해야 합니다.

이 문서는 단일 봇 Rails 앱을 기준으로, 다음 구조를 권장합니다.

1. 설정값 준비
2. initializer에서 client 생성과 webhook 등록
3. controller에서 Telegram POST 수신
4. `TelegramBot`이 payload 파싱부터 앱 로직까지 처리

## 1. 설정값 준비

다음 네 가지 설정값을 예시로 사용합니다.

- bot token
- webhook URL
- webhook secret token (선택)
- 부팅 시 webhook 등록 여부

예시:

```ruby
# credentials, ENV, Settings, config_for 중 편한 방식으로 준비하면 됩니다.
token = Rails.application.credentials.dig(:telegram, :bot_token)
webhook_url = Rails.application.routes.url_helpers.telegram_webhook_url(host: "https://example.com")
secret_token = Rails.application.credentials.dig(:telegram, :webhook_secret_token)
set_webhook_on_boot = true
```

- webhook URL은 Telegram이 실제로 호출할 공개 HTTPS URL이어야 합니다.
- webhook secret token은 선택값입니다. 설정하면 요청 헤더 `X-Telegram-Bot-Api-Secret-Token` 검증에 사용할 수 있습니다.
- 부팅 시 webhook 등록 여부는 앱 부팅 시 등록 로직을 실행할지 결정합니다.

## 2. Initializer에서 client 생성과 webhook 등록

예시 파일:

- `config/initializers/telegram_bot.rb`

```ruby
# frozen_string_literal: true

token = Rails.application.credentials.dig(:telegram, :bot_token)
client = Telegram::Bot::Client.new(token)

Rails.application.config.x.telegram_bot_client = client

if Rails.env.production?
  Rails.application.config.after_initialize do
    webhook_url = Rails.application.routes.url_helpers.telegram_webhook_url(host: "https://example.com")
    secret_token = Rails.application.credentials.dig(:telegram, :webhook_secret_token)

    info = client.api.get_webhook_info

    if info.url != webhook_url
      params = { url: webhook_url }
      params[:secret_token] = secret_token if secret_token.present?

      client.api.set_webhook(params)

      Rails.logger.info("Telegram webhook registered: #{webhook_url}")
    end
  end
end
```

주의할 점:

- `secret_token`은 선택값이므로 아예 사용하지 않아도 webhook은 동작합니다.
- `getWebhookInfo`는 secret token 자체를 돌려주지 않으므로 URL만 비교해서는 secret 변경을 감지할 수 없습니다.
- secret만 바뀌었다면 `delete_webhook` 후 다시 `set_webhook`을 호출하거나, 배포 시점에 수동으로 다시 등록해야 합니다.
- 부팅 시 외부 HTTP 호출이 부담스럽다면 같은 코드를 initializer가 아니라 deploy task나 rake task로 옮겨도 됩니다.

## 3. Route와 Controller에서 Telegram POST 받기

단일 봇 Rails 앱에서는 고정 path 하나를 두는 편이 단순합니다.

예시 route:

`config/routes.rb`

```ruby
Rails.application.routes.draw do
  post "/telegram/webhook", to: "telegram_webhooks#create"
end
```

예시 controller:

`app/controllers/telegram_webhooks_controller.rb`

```ruby
# frozen_string_literal: true

class TelegramWebhooksController < ActionController::API
  before_action :verify_telegram_secret!, if: :telegram_secret_configured?

  def create
    client = Rails.application.config.x.telegram_bot_client
    TelegramBot.new(client).process(request.raw_post)

    head :ok
  rescue StandardError => e
    Rails.logger.error("Telegram webhook error: #{e.class}: #{e.message}")
    status =
      case e
      when JSON::ParserError
        :bad_request
      else
        :internal_server_error
      end

    head status
  end

  private

  def verify_telegram_secret!
    expected = Rails.application.credentials.dig(:telegram, :webhook_secret_token)
    actual = request.headers["X-Telegram-Bot-Api-Secret-Token"]

    head :unauthorized unless ActiveSupport::SecurityUtils.secure_compare(actual.to_s, expected)
  end

  def telegram_secret_configured?
    Rails.application.credentials.dig(:telegram, :webhook_secret_token).present?
  end
end
```

기존 앱이 `ActionController::Base`를 사용 중이라면, CSRF 보호와 충돌하지 않도록 `skip_forgery_protection` 또는 동등한 예외 처리를 추가해야 합니다.

controller 내부 처리 순서는 이 정도로 고정하는 편이 좋습니다.

1. `request.raw_post`를 읽는다.
2. `TelegramBot` 같은 구체적인 객체에 raw payload를 넘긴다.
3. 성공 시 `200 OK`를 반환한다.

에러 처리 기본 정책:

- 잘못된 secret: `401 Unauthorized` (`secret_token`을 사용하는 경우)
- 잘못된 JSON: `400 Bad Request`
- 내부 예외: `500 Internal Server Error`

`500`을 반환하면 Telegram이 재시도할 수 있으므로, 일시 장애 상황에서는 이 동작이 유용할 수 있습니다.

## 4. 봇 로직은 구체적인 클래스로 두기

봇 로직은 컨트롤러 밖으로 빼고, `TelegramBot`처럼 구체적인 이름의 클래스를 하나 두는 편이 자연스럽습니다. 더 DHH스럽게 가려면 parsing과 이벤트 분기까지 이 클래스가 맡도록 두는 편이 깔끔합니다.

예시 위치:

- `app/models/telegram_bot.rb`
- 또는 `app/lib/telegram_bot.rb`

예시:

```ruby
# frozen_string_literal: true

class TelegramBot
  def initialize(client)
    @client = client
  end

  def process(raw_payload)
    payload = JSON.parse(raw_payload)
    update = Telegram::Bot::Types::Update.new(payload)

    case (event = update.current_message)
    when Telegram::Bot::Types::Message
      process_message(event)
    when Telegram::Bot::Types::CallbackQuery
      process_callback_query(event)
    end
  end

  private

  attr_reader :client

  def process_message(message)
    case message.text
    when "/start"
      client.api.send_message(
        chat_id: message.chat.id,
        text: "Hello, #{message.from.first_name}"
      )
    when "/stop"
      client.api.send_message(
        chat_id: message.chat.id,
        text: "Bye, #{message.from.first_name}"
      )
    end
  end

  def process_callback_query(callback_query)
    client.api.answer_callback_query(
      callback_query_id: callback_query.id
    )
  end
end
```

## 기존 polling 코드에서 옮기기

기존 long polling 예제는 보통 `bot.listen do |message| ... end` 안에 모든 로직이 들어갑니다. webhook으로 옮길 때는 그 블록 안의 로직을 `TelegramBot`으로 옮기면 됩니다.

기존:

```ruby
Telegram::Bot::Client.run(token) do |bot|
  bot.listen do |message|
    case message.text
    when "/start"
      bot.api.send_message(chat_id: message.chat.id, text: "Hello")
    end
  end
end
```

변경 후:

```ruby
Telegram::Bot::Client.run(token) do |bot|
  telegram_bot = TelegramBot.new(bot)

  bot.listen do |message|
    telegram_bot.process_event(message)
  end
end
```

webhook과 polling이 입력 형태만 다르고 결국 같은 이벤트를 처리한다면, `TelegramBot` 안에 작은 진입점 두 개만 두면 됩니다.

```ruby
def process_event(event)
  case event
  when Telegram::Bot::Types::Message
    process_message(event)
  when Telegram::Bot::Types::CallbackQuery
    process_callback_query(event)
  end
end

def process(raw_payload)
  payload = JSON.parse(raw_payload)
  update = Telegram::Bot::Types::Update.new(payload)
  process_event(update.current_message)
end
```

그러면 polling에서는 `process_event(message)`, webhook에서는 `process(request.raw_post)`를 호출하면 됩니다.

polling과 webhook을 동시에 운영하는 것은 Telegram 정책상 불가능하지만, 같은 봇 클래스를 재사용하는 구조는 마이그레이션과 테스트에 큰 도움이 됩니다.

## 운영 메모

- Telegram webhook은 공개 HTTPS 엔드포인트가 필요합니다.
- 로컬 개발에서는 ngrok 같은 터널링 도구로 `TELEGRAM_WEBHOOK_URL`을 임시로 노출할 수 있습니다.
- 배포 환경에서 webhook URL이 바뀌면 `set_webhook`이 다시 실행되도록 해야 합니다.
- 시간이 오래 걸리는 작업은 controller에서 직접 처리하지 말고 Active Job 같은 비동기 작업으로 넘기는 편이 안전합니다.

## Rails 테스트 시나리오

최소한 아래 시나리오는 검증하는 것을 권장합니다.

1. initializer의 등록 로직이 현재 webhook URL과 원하는 URL이 같으면 `set_webhook`을 호출하지 않는지
2. URL이 다를 때만 `set_webhook`을 호출하는지
3. `secret_token`을 설정한 경우 올바른 secret과 유효한 payload로 request를 보내면 `200`을 반환하는지
4. `secret_token`을 설정한 경우 secret이 틀리면 `401`을 반환하는지
5. `secret_token`을 사용하지 않는 경우 secret 헤더 없이도 `200`을 반환하는지
6. malformed JSON이면 `400`을 반환하는지
7. `TelegramBot#process`가 raw payload를 파싱해 기존 메시지 로직을 그대로 수행하는지

예시 request spec 형태:

```ruby
RSpec.describe "Telegram webhook", type: :request do
  let(:secret) { "test-secret" }
  let(:headers) do
    {
      "CONTENT_TYPE" => "application/json",
      "X-Telegram-Bot-Api-Secret-Token" => secret
    }
  end

  let(:payload) do
    {
      update_id: 123,
      message: {
        message_id: 456,
        date: Time.now.to_i,
        text: "/start",
        chat: { id: 1, type: "private" },
        from: { id: 2, is_bot: false, first_name: "Alice" }
      }
    }
  end

  it "returns ok" do
    post "/telegram/webhook", params: payload.to_json, headers: headers
    expect(response).to have_http_status(:ok)
  end
end
```
