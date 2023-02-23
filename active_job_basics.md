Основы Active Job
=================

Это руководство даст вам все, что нужно, чтобы начать создавать, ставить в очередь и выполнять фоновые задания.

После его прочтения, вы узнаете:

* Как создавать задания.
* Как ставить в очередь задания.
* Как запускать задания в фоне.
* Как асинхронно рассылать письма из вашего приложения.

--------------------------------------------------------------------------------

Что такое Active Job?
---------------------

Active Job - это фреймворк для объявления заданий и их запуска на разных бэкендах для очередей. Эти задания могут быть чем угодно, от регулярно запланированных чисток до списаний с карт или рассылок. В общем, всем, что может быть выделено в небольшие работающие части и запускаться параллельно.

Назначение Active Job
---------------------

Главным является то, что он обеспечивает, что у всех приложений на Rails имеется встроенная инфраструктура для заданий. Затем у нас могут появиться особенности фреймворка или других гемов, созданных на его основе, позволяющие не заботится об отличиях в API между различными исполнителями заданий, такими как Delayed Job и Resque. Подбор бэкенда для очередей станет более оперативной работой. Вы сможете переключаться между ними без необходимости переписывать свои задания.

NOTE: По умолчанию, Rails поставляется с асинхронной реализацией очереди, запускающей задания с помощью пула тредов внутри процесса. Задания будут запущены асинхронно, но любые задания в очереди будут потеряны при перезагрузке.

Создание задания
----------------

Этот раздел предоставляет пошаговое руководство к созданию задания и добавлению его в очередь.

### Создание задания

Active Job предоставляет генератор Rails для создания заданий. Следующая команда создаст задание в `app/jobs` (а также тестовый случай в `test/jobs`):

```bash
$ bin/rails generate job guests_cleanup
invoke  test_unit
create    test/jobs/guests_cleanup_job_test.rb
create  app/jobs/guests_cleanup_job.rb
```

Также можно создать задание, которое будет запущено в определенной очереди:

```bash
$ bin/rails generate job guests_cleanup --queue urgent
```

Если не хотите использовать генератор, можно создать файл очереди в `app/jobs`, просто убедитесь, что он наследуется от `ApplicationJob`.

Вот как выглядит задание:

```ruby
class GuestsCleanupJob < ApplicationJob
  queue_as :default

  def perform(*guests)
    # Сделать что-нибудь позже
  end
end
```

Отметьте, что можно определить `perform` с любым количеством аргументов.

Если у вас уже есть абстрактный класс, и его имя отличается от `ApplicationJob`, можно передать опцию `--parent`, чтобы обозначить, что вы желаете иной абстрактный класс:

```bash
$ bin/rails generate job process_payment --parent=payment_job
```

```ruby
class ProcessPaymentJob < PaymentJob
  queue_as :default

  def perform(*args)
    # Сделать что-нибудь позже
  end
end
```

### Помещение задания в очередь

Поместите задание в очередь с помощью [`perform_later`][] и, опционально, [`set`][]. Например, так:

```ruby
# Помещенное в очередь задание выполнится, как только освободится система очередей.
GuestsCleanupJob.perform_later guest
```

```ruby
# Помещенное в очередь задание выполнится завтра в полдень.
GuestsCleanupJob.set(wait_until: Date.tomorrow.noon).perform_later(guest)
```

```ruby
# Помещенное в очередь задание выполнится через неделю.
GuestsCleanupJob.set(wait: 1.week).perform_later(guest)
```

```ruby
# `perform_now` и `perform_later` вызывают `perform`, поэтому
# можно передать столько аргументов, сколько определено в последнем.
GuestsCleanupJob.perform_later(guest1, guest2, filter: 'some_filter')
```

Вот и все!

[`perform_later`]: https://api.rubyonrails.org/classes/ActiveJob/Enqueuing/ClassMethods.html#method-i-perform_later
[`set`]: https://api.rubyonrails.org/classes/ActiveJob/Core/ClassMethods.html#method-i-set

Выполнение заданий
------------------

Чтобы поместить задание в очередь и выполнить его в production, необходимо настроить бэкенд для очереди, т.е. нужно решить, какую стороннюю библиотеку для очереди Rails будет использовать. Rails предоставляет только внутрипроцессную систему очереди, хранящую задания в памяти. Если процесс упадет, или машина будет перезагружена, тогда в асинхронном бэкенде по умолчанию все оставшиеся задания будут потеряны. Это может быть нормальным для маленьких приложений или некритичных заданий, но для большей части серьезных приложений нужно подобрать персистентный бэкенд.

### Бэкенды

У Active Job есть встроенные адаптеры для различных бэкендов очередей (Sidekiq, Resque, Delayed Job и другие). Чтобы получить актуальный список адаптеров, обратитесь к документации API по [`ActiveJob::QueueAdapters`][].

[`ActiveJob::QueueAdapters`]: https://api.rubyonrails.org/classes/ActiveJob/QueueAdapters.html

### Настройка бэкенда

Настроить бэкенд — это просто с помощью [`config.active_job.queue_adapter`]:

```ruby
# config/application.rb
module YourApp
  class Application < Rails::Application
    # Убедитесь, что гем адаптера добавлен в Gemfile, и что выполнены
    # инструкции по установке и развертыванию адаптера.
    config.active_job.queue_adapter = :sidekiq
  end
end
```

Также можно настроить бэкенд для отдельного задания:

```ruby
class GuestsCleanupJob < ApplicationJob
  self.queue_adapter = :resque
  # ...
end

# Теперь ваше задание будет использовать `resque` в качестве адаптера бэкенда очереди,
# переопределяя тот, что был настроен в `config.active_job.queue_adapter`.
```

[`config.active_job.queue_adapter`]: /configuring#config-active-job-queue-adapter

### Запуск бэкенда

Поскольку задания запускаются параллельно с вашим Rails приложением, большинство библиотек для работы с очередями требуют запуска специфичного для библиотеки сервиса очередей (помимо старта Rails приложения) для обработки заданий. Обратитесь к документации по библиотеке за инструкциями по запуску бэкенда очереди.

Вот неполный список документации:

- [Sidekiq](https://github.com/mperham/sidekiq/wiki/Active-Job)
- [Resque](https://github.com/resque/resque/wiki/ActiveJob)
- [Sneakers](https://github.com/jondot/sneakers/wiki/How-To:-Rails-Background-Jobs-with-ActiveJob)
- [Sucker Punch](https://github.com/brandonhilkert/sucker_punch#active-job)
- [Queue Classic](https://github.com/QueueClassic/queue_classic#active-job)
- [Delayed Job](https://github.com/collectiveidea/delayed_job#active-job)
- [Que](https://github.com/que-rb/que#additional-rails-specific-setup)
- [Good Job](https://github.com/bensheldon/good_job#readme)

Очереди
-------

Большая часть адаптеров поддерживает несколько очередей. С помощью Active Job можно запланировать, что задание будет выполнено в определенной очереди, с помощью [`queue_as`][]:

```ruby
class GuestsCleanupJob < ApplicationJob
  queue_as :low_priority
  # ...
end
```

Можно задать префикс для имени очереди для всех заданий с помощью [`config.active_job.queue_name_prefix`][] в `application.rb`:

```ruby
# config/application.rb
module YourApp
  class Application < Rails::Application
    config.active_job.queue_name_prefix = Rails.env
  end
end
```

```ruby
# app/jobs/guests_cleanup_job.rb
class GuestsCleanupJob < ApplicationJob
  queue_as :low_priority
  # ...
end

# Теперь ваше задание запустится в очереди production_low_priority в среде
# production и в staging_low_priority в среде staging
```

Также можно настроить префикс на уровне задания.

```ruby
class GuestsCleanupJob < ApplicationJob
  queue_as :low_priority
  self.queue_name_prefix = nil
  # ...
end

# Теперь очередь задания не будет иметь префикс, переопределяя то,
# что было настроено в `config.active_job.queue_name_prefix`.
```

Разделитель префикса имени очереди по умолчанию '\_'. Его можно изменить, установив [`config.active_job.queue_name_delimiter`][] в `application.rb`:

```ruby
# config/application.rb
module YourApp
  class Application < Rails::Application
    config.active_job.queue_name_prefix = Rails.env
    config.active_job.queue_name_delimiter = '.'
  end
end
```

```ruby
# app/jobs/guests_cleanup_job.rb
class GuestsCleanupJob < ApplicationJob
  queue_as :low_priority
  # ...
end

# Теперь ваше задание запустится в очереди production.low_priority в среде
# production и в staging.low_priority в среде staging
```

Если хотите больше контроля, в какой очереди задание будет запущено, можно передать опцию `:queue` в `set`:

```ruby
MyJob.set(queue: :another_queue).perform_later(record)
```

Чтобы контролировать очередь на уровне задания, можно передать блок в `queue_as`. Блок будет выполнен в контексте задания (таким образом, у него будет доступ к `self.arguments`), и он должен вернуть имя очереди:

```ruby
class ProcessVideoJob < ApplicationJob
  queue_as do
    video = self.arguments.first
    if video.owner.premium?
      :premium_videojobs
    else
      :videojobs
    end
  end

  def perform(video)
    # Делаем обработку видео
  end
end
```

```ruby
ProcessVideoJob.perform_later(Video.last)
```

NOTE: Убедитесь, что ваш бэкенд для очередей "слушает" имя вашей очереди. Для некоторых бэкендов необходимо указать очереди, которые нужно слушать.

[`config.active_job.queue_name_delimiter`]: /configuring#config-active-job-queue-name-delimiter
[`config.active_job.queue_name_prefix`]: /configuring#config-active-job-queue-name-prefix
[`queue_as`]: https://api.rubyonrails.org/classes/ActiveJob/QueueName/ClassMethods.html#method-i-queue_as

Колбэки
-------

Active Job предоставляет хуки для включения логики на протяжение жизненного цикла задания. Подобно другим колбэкам в Rails, можно реализовывать колбэки как обычные методы и использовать макрос-метод класса, чтобы зарегистрировать их в качестве колбэков:

```ruby
class GuestsCleanupJob < ApplicationJob
  queue_as :default

  around_perform :around_cleanup

  def perform
    # Отложенное задание
  end

  private
    def around_cleanup
      # Делаем что-то перед perform
      yield
      # Делаем что-то после perform
    end
end
```

Макрос-методы класса также могут принимать блок. Рассмотрите возможность использования этого макроса, если код внутри блока настолько короток, что он помещается в одну строчку. Например, можно отправлять показатели для каждого помещенного в очередь задания.

```ruby
class ApplicationJob < ActiveJob::Base
  before_enqueue { |job| $statsd.increment "#{job.class.name.underscore}.enqueue" }
end
```

### Доступные колбэки

* [`before_enqueue`][]
* [`around_enqueue`][]
* [`after_enqueue`][]
* [`before_perform`][]
* [`around_perform`][]
* [`after_perform`][]

[`before_enqueue`]: https://api.rubyonrails.org/classes/ActiveJob/Callbacks/ClassMethods.html#method-i-before_enqueue
[`around_enqueue`]: https://api.rubyonrails.org/classes/ActiveJob/Callbacks/ClassMethods.html#method-i-around_enqueue
[`after_enqueue`]: https://api.rubyonrails.org/classes/ActiveJob/Callbacks/ClassMethods.html#method-i-after_enqueue
[`before_perform`]: https://api.rubyonrails.org/classes/ActiveJob/Callbacks/ClassMethods.html#method-i-before_perform
[`around_perform`]: https://api.rubyonrails.org/classes/ActiveJob/Callbacks/ClassMethods.html#method-i-around_perform
[`after_perform`]: https://api.rubyonrails.org/classes/ActiveJob/Callbacks/ClassMethods.html#method-i-after_perform

Action Mailer
-------------

Одним из обычных заданий в современном веб-приложении является рассылка писем за пределами цикла запроса-отклика, чтобы пользователь не ждал. Active Job интегрируется с Action Mailer, поэтому рассылать письма асинхронно очень просто:

```ruby
# Если хотите отправить письмо сейчас, используйте #deliver_now
UserMailer.welcome(@user).deliver_now

# Если хотите отправить письмо через Active Job, используйте #deliver_later
UserMailer.welcome(@user).deliver_later
```

NOTE: Использование асинхронной очереди из задач Rake (например, для отправки электронной почты с помощью `.deliver_later`), как правило, не будет работать, потому что Rake, вероятно, завершится, в результате чего пул тредов внутри процесса будет удален до того, как любой/все из `.deliver_later` писем будут обработаны. Чтобы избежать этой проблемы, используйте `.deliver_now` или запустите персистентную очередь в development режиме.

Интернационализация
-------------------

Каждое задание использует настройку `I18n.locale` при создании. Это полезно, если вы отправляете письма асинхронно:

```ruby
I18n.locale = :eo

UserMailer.welcome(@user).deliver_later # Email будет локализован в Эсперанто.
```

Поддерживаемые типы аргументов
------------------------------

ActiveJob по умолчанию поддерживает следующие типы аргументов:

- Базовые типы (`NilClass`, `String`, `Integer`, `Float`, `BigDecimal`, `TrueClass`, `FalseClass`)
- `Symbol`
- `Date`
- `Time`
- `DateTime`
- `ActiveSupport::TimeWithZone`
- `ActiveSupport::Duration`
- `Hash` (Ключи должны быть типа `String` или `Symbol`)
- `ActiveSupport::HashWithIndifferentAccess`
- `Array`
- `Range`
- `Module`
- `Class`

### GlobalID

Active Job поддерживает [GlobalID](https://github.com/rails/globalid/blob/master/README.md) для параметров. Это позволяет передавать объекты Active Record в ваши задания, вместо пар класс/id, которые нужно затем десериализовать вручную. Раньше задания выглядели так:

```ruby
class TrashableCleanupJob < ApplicationJob
  def perform(trashable_class, trashable_id, depth)
    trashable = trashable_class.constantize.find(trashable_id)
    trashable.cleanup(depth)
  end
end
```

Теперь можно просто сделать так:

```ruby
class TrashableCleanupJob < ApplicationJob
  def perform(trashable, depth)
    trashable.cleanup(depth)
  end
end
```

Это работает с любым классом, в который подмешан `GlobalID::Identification`, который по умолчанию был подмешан в классы Active Record.

### Сериализаторы

Можно расширить список поддерживаемых типов для аргументов. Для этого необходимо определить свой собственный сериализатор.

```ruby
# app/serializers/money_serializer.rb
class MoneySerializer < ActiveJob::Serializers::ObjectSerializer
  # Проверяем, должен ли argument быть сериализован с использованием этого сериализатора.
  def serialize?(argument)
    argument.is_a? Money
  end

  # Преобразование объекта к более простому представителю, используя поддерживаемые типы объектов.
  # Рекомендуемым представителем является хэш с определенным ключом. Ключи могут быть только базового типа.
  # Необходимо вызвать `super`, чтобы добавить собственный тип сериализатора в хэш.
  def serialize(money)
    super(
      "amount" => money.amount,
      "currency" => money.currency
    )
  end

  # Преобразование сериализованного значения в надлежащий объект.
  def deserialize(hash)
    Money.new(hash["amount"], hash["currency"])
  end
end
```

и добавить этот сериализатор в список:

```ruby
# config/initializers/custom_serializers.rb
Rails.application.config.active_job.custom_serializers << MoneySerializer
```

Отметьте, что автозагрузка перезагружаемого кода в течение инициализации не поддерживается. Поэтому рекомендуется настраивать сериализаторы, чтобы они загружались лишь однажды, то есть изменяя `config/application.rb` таким образом:

```ruby
# config/application.rb
module YourApp
  class Application < Rails::Application
    config.autoload_once_paths << Rails.root.join('app', 'serializers')
  end
end
```

Исключения
----------

Исключения, вызванные в течение исполнения задания, могут быть обработаны с помощью [`rescue_from`][]:

```ruby

class GuestsCleanupJob < ApplicationJob
  queue_as :default

  rescue_from(ActiveRecord::RecordNotFound) do |exception|
    # Сделать что-то с этим исключением
  end

  def perform
    # Отложенное задание
  end
end
```

Если исключение от задания не будет поймано, тогда задание будет помечено как "неудачное".

[`rescue_from`]: https://api.rubyonrails.org/classes/ActiveSupport/Rescuable/ClassMethods.html#method-i-rescue_from

### Повторная отправка или отмена неудачных заданий

Неудачное задание не будет повторено, если не настроено обратное.

Возможно повторить отправку или отменить неудачное задание, с помощью [`retry_on`] или [`discard_on`], соответственно. Например:

```ruby
class RemoteServiceJob < ApplicationJob
  retry_on CustomAppException # по умолчанию, ожидание: 3 сек., попыток: 5

  discard_on ActiveJob::DeserializationError

  def perform(*args)
    # Может быть вызвано CustomAppException или ActiveJob::DeserializationError
  end
end
```

[`discard_on`]: https://api.rubyonrails.org/classes/ActiveJob/Exceptions/ClassMethods.html#method-i-discard_on
[`retry_on`]: https://api.rubyonrails.org/classes/ActiveJob/Exceptions/ClassMethods.html#method-i-retry_on

### Десериализация

GlobalID позволяет сериализовать полностью объекты Active Record, переданные в `#perform`.

Если переданная запись была удалена после того, как задание было помещено в очередь, но до того, как метод `#perform` был вызван, Active Job вызовет исключение [`ActiveJob::DeserializationError`][].

[`ActiveJob::DeserializationError`]: https://api.rubyonrails.org/classes/ActiveJob/DeserializationError.html

Тестирование заданий
--------------------

Подробные инструкции о том, как тестировать ваши задания, можно найти в руководстве [Тестирование приложений на Rails](testing#jobs-testing).
