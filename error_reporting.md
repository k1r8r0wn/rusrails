Отчет об ошибках в приложениях Rails
====================================

Это руководство представляет способы управления исключениями, которые случаются в приложениях Ruby on Rails.

После прочтения этого руководства, вы узнаете:

* Как использовать репортер об ошибках Rails, чтобы отлавливать и отчитываться об ошибках.
* Как создавать пользовательских получателей для вашего отчитывающегося об ошибке сервиса.

--------------------------------------------------------------------------------

Отчет об ошибке
---------------

[Репортер об ошибках](https://api.rubyonrails.org/classes/ActiveSupport/ErrorReporter.html) Rails предоставляет стандартных способ собирать исключения, случающиеся в вашем приложении, и сообщать о них предпочитаемому вами сервису или месту.

Репортер об ошибках нацелен заменить шаблонный код обработки ошибки, наподобие:

```ruby
begin
  do_something
rescue SomethingIsBroken => error
  MyErrorReportingService.notify(error)
end
```

с помощью последовательного интерфейса:

```ruby
Rails.error.handle(SomethingIsBroken) do
  do_something
end
```

Rails оборачивает все выполнения (такие как запросы HTTP, задачи и вызовы `rails runner`) в репортер об ошибках, поэтому любые необработанные ошибки, вызванные в вашем приложении, автоматически будут сообщены вашему сервису, отчитывающемуся об ошибке, через их получателей.

Это означает, что сторонним библиотекам, отчитывающимся об ошибке, больше не нужно вставлять промежуточную программу Rack или делать какие-либо Monkey patch, чтобы захватить необработанные исключения. Библиотеки, использующие ActiveSupport, также могут это использовать, чтобы ненавязчиво отчитываться о предупреждениях, которые раньше терялись в логах.

Использование репортера об ошибках Rails не обязательное. Все иные средства отлова ошибок все еще работают.

### Подписка на репортер

Чтобы использовать репортер об ошибках, необходим _получатель_. Получателем может быть любым объектом с методом `report`. Когда в вашем приложении происходит ошибка, или вызывается вручную, репортер об ошибках Rails вызовет этот метод с объектом ошибки и некоторыми опциями.

Некоторые библиотеки для отчета об ошибках, такие как [Sentry](https://github.com/getsentry/sentry-ruby/blob/e18ce4b6dcce2ebd37778c1e96164684a1e9ebfc/sentry-rails/lib/sentry/rails/error_subscriber.rb) и [Honeybadger](https://docs.honeybadger.io/lib/ruby/integration-guides/rails-exception-tracking/), автоматически регистрируют для вас получателя. За подробностями обратитесь к документации вашего поставщика.

Также можно создать пользовательского получателя. Например:

```ruby
# config/initializers/error_subscriber.rb
class ErrorSubscriber
  def report(error, handled:, severity:, context:, source: nil)
    MyErrorReportingService.report_error(error, context: context, handled: handled, level: severity)
  end
end
```

После определения класса получателя, зарегистрируйте его, вызвав метод [`Rails.error.subscribe`](https://api.rubyonrails.org/classes/ActiveSupport/ErrorReporter.html#method-i-subscribe):

```ruby
Rails.error.subscribe(ErrorSubscriber.new)
```

Можно зарегистрировать столько получателей, сколько желаете. Rails вызовет их по очереди, в порядке, в котором они зарегистрированы.

Note: Репортер об ошибках Rails всегда будет запускать зарегистрированные получатели, независимо от среды. Однако, многие сервисы отчета об ошибках сообщают об ошибках по умолчанию только в production. Вам следует настроить и протестировать ваши настройки в разных средах по необходимости.

### Использование репортера об ошибках

Есть три способа использования репортера об ошибках:

#### Отчет и проглатывание ошибок

[`Rails.error.handle`](https://api.rubyonrails.org/classes/ActiveSupport/ErrorReporter.html#method-i-handle) отчитается о любой ошибке, вызванной в блоке. Затем он **проглотит** ошибку, и остальной ваш код вне блока будет продолжаться, как обычно.

```ruby
result = Rails.error.handle do
  1 + '1' # вызовет TypeError
end
result # => nil
1 + 1 # Это будет выполнено
```

Если в блоке не вызвана ошибка, `Rails.error.handle` возвратит результат блока, в противном случае вернет `nil`. Это можно переопределить, предоставив `fallback`:

```ruby
user = Rails.error.handle(fallback: -> { User.anonymous }) do
  User.find_by(params[:id])
end
```

#### Отчет и повторный вызов ошибок

[`Rails.error.record`](https://api.rubyonrails.org/classes/ActiveSupport/ErrorReporter.html#method-i-record) отчитается об ошибках всем зарегистрированным получателям, а затем повторно вызовет ошибку, что означает, что остальной ваш код не будет выполнен.

```ruby
Rails.error.record do
  1 + '1' # вызовет TypeError
end
1 + 1 # Это не будет выполнено
```

Если в блоке не вызвана ошибка, `Rails.error.record` возвратит результат блока.

#### Отчет об ошибках вручную

Также можно вручную отчитаться об ошибках, вызвав [`Rails.error.report`](https://api.rubyonrails.org/classes/ActiveSupport/ErrorReporter.html#method-i-report):

```ruby
begin
  # код
rescue StandardError => e
  Rails.error.report(e)
end
```

Любые переданные опции будут переданы получателям ошибки.

### Опции отчета об ошибки

Все 3 отчитывающихся API (`#handle`, `#record` и `#report`) поддерживают следующие опции, которые затем передаются во все зарегистрированные получатели:

- `handled`: `Boolean` для обозначения, была ли обработана ошибка. По умолчанию `true`. `#record` устанавливает ей `false`.
- `severity`: `Symbol`, описывающий суровость ошибки. Ожидаемые значения: `:error`, `:warning` и `:info`. `#handle` устанавливает ей `:warning`, в то время как `#record` устанавливает ей `:error`.
- `context`: `Hash` для предоставления больше контекста об ошибке, наподобие подробностей о запросе или пользователе
- `source`: `String` об источнике ошибки. Источник по умолчанию `"application"`. Ошибки, вызываемые внутренними библиотеками, могут устанавливать другие источники; библиотека кэша Redis может использовать `"redis_cache_store.active_support"`, например. Ваш получатель может использовать источник, чтобы игнорировать ошибки, в которой он не заинтересован.

```ruby
Rails.error.handle(context: {user_id: user.id}, severity: :info) do
  # ...
end
```

### Фильтрация по классам ошибок

С помощью `Rails.error.handle` и `Rails.error.record` также можно выбрать, чтобы отчитываться только об ошибках определенных классов. Например:

```ruby
Rails.error.handle(IOError) do
  1 + '1' # вызовет TypeError
end
1 + 1 # TypeErrors это не IOError, поэтому это *не* будет выполнено
```

Тут `TypeError` не будет отловлено репортером об ошибках Rails. Будет только отчет об экземплярах `IOError` и ее потомков. Любые другие ошибки будут вызваны, как обычно.

### Установка глобального контекста

В дополнение к установке контекста с помощью опции `context`, можно использовать [`#set_context`](https://api.rubyonrails.org/classes/ActiveSupport/ErrorReporter.html#method-i-set_context) API. Например:

```ruby
Rails.error.set_context(section: "checkout", user_id: @user.id)
```

Любой контекст, установленный таким образом, будет объединена с опцией `context`

```ruby
Rails.error.set_context(a: 1)
Rails.error.handle(context: { b: 2 }) { raise }
# Контекст отчета будет: {:a=>1, :b=>2}
Rails.error.handle(context: { b: 3 }) { raise }
# Контекст отчета будет: {:a=>1, :b=>3}
```

### Для библиотек

Библиотеки для отчета об ошибках могут регистрировать своих получателей в `Railtie`:

```ruby
module MySdk
  class Railtie < ::Rails::Railtie
    initializer "error_subscribe.my_sdk" do
      Rails.error.subscribe(MyErrorSubscriber.new)
    end
  end
end
```

Если вы зарегистрировали получателя ошибки, но все еще имеете другие механизмы для ошибок, наподобие промежуточной программы Rack, может получиться, что ошибки сообщаются несколько раз. Следует либо убрать ваши другие механизмы, либо исправить функционал вашего отчета, чтобы он пропускал сообщения об исключении, которое он уже видел.
