Рекомендации по документированию API
====================================

Это руководство документирует рекомендации к документации Ruby on Rails API.

После его прочтения, вы узнаете:

* Как писать эффективную прозу для целей документирования.
* Стилистические рекомендации для документирования различного кода на Ruby.

--------------------------------------------------------------------------------

RDoc
----

[Документация Rails API](https://api.rubyonrails.org) генерируется с помощью [RDoc](https://ruby.github.io/rdoc/). Чтобы ее сгенерировать, убедитесь, что вы в корневой директории rails, запустите `bundle install` и выполните:

```bash
$ bundle exec rake rdoc
```

Итоговые файлы HTML будут в директории ./doc/rdoc.

Обратитесь к документации RDoc за помощью с [разметкой](https://ruby.github.io/rdoc/RDoc/Markup.html), а также примите во внимание эти [дополнительные директивы](https://ruby.github.io/rdoc/RDoc/Parser/Ruby.html).

Ссылки
------

Документация Rails API не означает, что она будет просматриваемая на GitHub, и поэтому ссылки должны использовать разметку RDoc [`link`](https://ruby.github.io/rdoc/RDoc/Markup.html#class-RDoc::Markup-label-Links), относительно текущего API.

Это потому, что разница между GitHub Markdown и сгенерированным RDoc в том, что это публикуется на [api.rubyonrails.org](https://api.rubyonrails.org) и [edgeapi.rubyonrails.org](https://edgeapi.rubyonrails.org).

Например, мы используем `[link:classes/ActiveRecord/Base.html]` для создания ссылки на класс `ActiveRecord::Base`, генерируемой RDoc.

Это предпочтительней, чем абсолютный URL, такой как `[https://api.rubyonrails.org/classes/ActiveRecord/Base.html]`, который может перевести читателя на другую версию документации (например, edgeapi.rubyonrails.org).

(wording) Формулировки
----------------------

Пишите простые, декларативные предложения. Краткость — сестра таланта.

Пишите в настоящем времени: "Returns a hash that...", а не "Returned a hash that...", или "Will return a hash that...".

Начинайте комментарии с большой буквы. Следуйте обычным правилам пунктуации:

```ruby
# Declares an attribute reader backed by an internally-named
# instance variable.
def attr_internal_reader(*attrs)
  # ...
end
```

Доносите до читателя правильный способ делать вещи, явно или неявно. Используйте идиомы, рекомендованные в последней версии. По необходимости перегруппируйте разделы, чтобы подчеркнуть предпочтительные подходы, и так далее. Документация должна быть моделью для лучшей практики и канонического, современного использования Rails.

Документация должна быть краткой, но всеобъемлющей. Исследуйте и документируйте крайние случаи. Что произойдет, если модуль анонимный? А что, если коллекция пустая? А если аргумент nil?

Правильные имена компонентов Rails имеют пробел между словами, например "Active Support". `ActiveRecord` это модуль на Ruby, в то время как Active Record — это ORM. Вся документация Rails должна последовательно ссылаться на компоненты Rails по их правильным именам.

Пишите имена правильно: Arel, minitest, RSpec, HTML, MySQL, JavaScript, ERB, Hotwire. Когда сомневаетесь, взгляните на какой-нибудь авторитетный источник, например, на их официальную документацию.

Используйте артикль "an" для "SQL", как в "an SQL statement". А также "an SQLite database".

Предпочитайте формулировки, избегающие использование "you" и "your". Например, вместо

```markdown
If you need to use `return` statements in your callbacks, it is recommended that you explicitly define them as methods.
```

используйте такую стилистику:

```markdown
If `return` is needed it is recommended to explicitly define a method.
```

Тем не менее, при использовании местоимений для ссылки на гипотетическую личность, такую как "a user with a session cookie", должны быть использованы гендерно нейтральные местоимения (they/their/them). Вместо:

* he или she... используйте they.
* him или her... используйте them.
* his или her... используйте their.
* his или hers... используйте theirs.
* himself или herself... используйте themselves.

(english) Английский язык
-------------------------

Пожалуйста, используйте американский английский (*color*, *center*, *modularize* и т.д.). Обратитесь к [списку различий в написании американских и британских английских слов](https://en.wikipedia.org/wiki/American_and_British_English_spelling_differences).

Оксфордская запятая
-------------------

Пожалуйста, используйте [оксфордскую запятую](https://ru.wikipedia.org/wiki/Серийная_запятая) ("red, white, and blue" вместо "red, white and blue").

(example-code) Пример кода
--------------------------

Выбирайте осмысленные примеры, отображающие и покрывающие как основы, так и интересные моменты или подводные камни.

Используйте два пробела для отступа кусков кода — это означает, для целей разметки, два пробела с учетом левого отступа. Сами примеры должны следовать [соглашениям по программированию Rails](/contributing_to_ruby_on_rails#follow-the-coding-conventions).

Короткие документы не нуждаются в явной метке "Examples" для представления примеров кода; они идут следующим параграфом:

```ruby
# Converts a collection of elements into a formatted string by
# calling +to_s+ on all elements and joining them.
#
#   Blog.all.to_fs # => "First PostSecond PostThird Post"
```

С другой стороны, большие куски структурированной документации могут иметь отдельный раздел "Examples":

```ruby
# ==== Examples
#
#   Person.exists?(5)
#   Person.exists?('5')
#   Person.exists?(name: "David")
#   Person.exists?(['name LIKE ?', "%#{query}%"])
```

Результаты выражений следуют за ними и представляются с помощью "# => ", выравненными по вертикали:

```ruby
# For checking if a fixnum is even or odd.
#
#   1.even? # => false
#   1.odd?  # => true
#   2.even? # => true
#   2.odd?  # => false
```

Если строчка слишком длинная, комментарий может быть помещен в следующей строчке:

```ruby
#   label(:article, :title)
#   # => <label for="article_title">Title</label>
#
#   label(:article, :title, "A short title")
#   # => <label for="article_title">A short title</label>
#
#   label(:article, :title, "A short title", class: "title_label")
#   # => <label for="article_title" class="title_label">A short title</label>
```

Избегайте использования любых выводящих методов, таких как `puts` или `p`, для этой цели.

С другой стороны, обычные комментарии не используют стрелку:

```ruby
#   polymorphic_url(record)  # same as comment_url(record)
```

Булевы значения
---------------

В предикатах и флажках предпочитайте документировать булеву семантику над остальными значениями.

Когда "true" или "false" используются так, как определены в Ruby, используйте обычный шрифт. Синглтонам `true` и `false` необходим моноширинный шрифт. Пожалуйста, избегайте терминов, таких как "truthy", Ruby определяет, что такое true и false в языке, и поэтому эти слова имеют техническое значение и не нуждаются в замене.

Как правило, не документируйте синглтоны, кроме случая, когда это абсолютно необходимо. Это предотвращает искусственные конструкции, наподобие `!!` или тернарного оператора, позволяет рефакторинг, и коду не нужно полагаться на точные значения, возвращаемыми в этой реализации.

Например, в:

```markdown
`config.action_mailer.perform_deliveries` specifies whether mail will actually be delivered and is true by default
```

пользователю не нужно знать, какое действительное значение по умолчанию у этого флажка, поэтому мы документируем только его булеву семантику.

Пример с предикатом:

```ruby
# Returns true if the collection is empty.
#
# If the collection has been loaded
# it is equivalent to <tt>collection.size.zero?</tt>. If the
# collection has not been loaded, it is equivalent to
# <tt>!collection.exists?</tt>. If the collection has not already been
# loaded and you are going to fetch the records anyway it is better to
# check <tt>collection.length.zero?</tt>.
def empty?
  if loaded?
    size.zero?
  else
    @target.blank? && !scope.exists?
  end
end
```

API осторожен, чтобы не обещать какое-либо конкретное значение, у метода есть семантика предиката, этого достаточно.

(file-names) Имена файлов
-------------------------

Как правило, используйте имена файлов относительно корня приложения:

```
config/routes.rb            # ДА
routes.rb                   # НЕТ
RAILS_ROOT/config/routes.rb # НЕТ
```

(fonts) Шрифты
--------------

### Моноширинный шрифт

Используйте моноширинные шрифты для:

* Констант, в частности имен классов и модулей.
* Имен методов.
* Литералов, таких как `nil`, `false`, `true`, `self`.
* Символов.
* Параметров методов.
* Имен файлов.

```ruby
class Array
  # Calls +to_param+ on all its elements and joins the result with
  # slashes. This is used by +url_for+ in Action Pack.
  def to_param
    collect { |e| e.to_param }.join '/'
  end
end
```

WARNING: Использование `+...+` для моноширинного шрифта работает только с простым содержимым, таким как обычные имена методов, символы, пути (с обратными слэшами) и так далее. Для всего остального используйте `<tt>...</tt>`, особенно для имен классов или модулей в пространстве имен, как в `<tt>ActiveRecord::Base</tt>`.

Можно быстро протестировать результат RDoc с помощью следующей команды:

```bash
$ echo "+:to_param+" | rdoc --pipe
# => <p><code>:to_param</code></p>
```

### Обычный шрифт

Когда "true" и "false" это английские слова, в отличие от ключевых слов Ruby, используйте обычный шрифт:

```ruby
# Runs all the validations within the specified context.
# Returns true if no errors are found, false otherwise.
#
# If the argument is false (default is +nil+), the context is
# set to <tt>:create</tt> if <tt>new_record?</tt> is true,
# and to <tt>:update</tt> if it is not.
#
# Validations with no <tt>:on</tt> option will run no
# matter the context. Validations with # some <tt>:on</tt>
# option will only run in the specified context.
def valid?(context = nil)
  # ...
end
```

Описательные списки
-------------------

В списках опций, параметров и т.д. используйте дефис между элементом и его описанием (читается лучше, чем двоеточие, так как опции обычно символы):

```ruby
# * <tt>:allow_nil</tt> - Skip validation if attribute is +nil+.
```

Описание начинается с заглавной буквы и заканчивается точкой — это стандартный английский.

Динамически создаваемые методы
------------------------------

Методы, созданные с помощью `(module|class)_eval(STRING)`, имеют справа комментарий с экземпляром сгенерированного кода. Этот комментарий отделен двумя пробелами от шаблона:

```ruby
for severity in Severity.constants
  class_eval <<-EOT, __FILE__, __LINE__ + 1
    def #{severity.downcase}(message = nil, progname = nil, &block)  # def debug(message = nil, progname = nil, &block)
      add(#{severity}, message, progname, &block)                    #   add(DEBUG, message, progname, &block)
    end                                                              # end
                                                                     #
    def #{severity.downcase}?                                        # def debug?
      #{severity} >= @level                                          #   DEBUG >= @level
    end                                                              # end
  EOT
end
```

Если результирующие строчки получаются слишком длинными, скажем 200 символов и больше, поместите комментарий над вызовом:

```ruby
# def self.find_by_login_and_activated(*args)
#   options = args.extract_options!
#   ...
# end
self.class_eval %{
  def self.#{method_id}(*args)
    options = args.extract_options!
    ...
  end
}
```

Видимость методов
-----------------

При написании документации для Rails, важно понимать различие между публичным пользовательским API и внутренним API.

Rails, как и многие библиотеки, использует ключевое слово private из Ruby для определения внутреннего API. Однако, публичный API следует немного отличающимся соглашениям. Вместо предположения, что все публичные методы разработаны для пользователя, Rails использует директиву `:nodoc:` для аннотирования таких методов как внутреннего API.

Это означает, что в Rails имеются методы с видимостью `public`, не предназначенные для пользователя.

Примером этого является `ActiveRecord::Core::ClassMethods#arel_table`:

```ruby
module ActiveRecord::Core::ClassMethods
  def arel_table # :nodoc:
    # do some magic..
  end
end
```

Если вы подумаете: "этот метод выглядит публичным методом класса для `ActiveRecord::Core`", то будете правы. Но фактически, основная команда Rails не хочет, чтобы пользователи полагались на этот метод. Поэтому они пометили его как `:nodoc:`, и он убран из публичной документации. Обоснованием этого является необходимость позволить команде изменить эти методы в соответствии с их внутренними потребностями между релизами, по их усмотрению. Может измениться имя этого метода, или возвращаемое значение, или может исчезнуть весь этот класс; нет гарантии, и поэтому вы не должны зависеть от этого API в ваших плагинах или приложениях. В противном случае, есть риск, что приложение или гем сломаются при апгрейде до более нового релиза Rails.

В качестве контрибьютора важно думать, предназначен ли этот API для конечного пользователя. Команда Rails стремится не вносить какие-либо критические изменения в публичный API между релизами без прохождения полного цикла устаревания. Рекомендуется, чтобы вы помечали `:nodoc:` любые ваши внутренние методы/классы, за исключением тех, которые уже приватные (имеется в виду видимость), в этом случае они внутренние по умолчанию. Как только API стабилизируется, видимость может измениться, но изменение публичного API намного сложнее из-за обратной совместимости.

Классы или модули, помеченные `:nodoc:`, показывают, что все методы являются внутренним API, и никогда не должны использоваться непосредственно.

Итак, команда Rails использует `:nodoc:`, чтобы пометить публично видимые методы и классы для внутреннего использования; изменения видимости в API должны сначала тщательно рассматриваться и обсуждаться в пул-реквесте.

Относительно стека Rails
------------------------

При документировании частей Rails API важно помнить все кусочки, составляющие стек Rails.

Это означает, что поведение может измениться в зависимости от области видимости или контекста метода или класса, который вы пытаетесь документировать.

В различных местах имеется различное поведение, если принимать во внимание полный стек, одним из примеров является `ActionView::Helpers::AssetTagHelper#image_tag`:

```ruby
# image_tag("icon.png")
#   # => <img src="/assets/icon.png" />
```

Хотя поведением по умолчанию для `#image_tag` является всегда возвращать `/images/icon.png`, учитывая полный стек Rails (включая Asset Pipeline), мы можем увидеть вышеуказанный результат.

Мы беспокоимся только о поведении при использовании полного стека Rails по умолчанию.

В этом случае мы хотим документировать поведение этого _фреймворка_, а не просто этого отдельного метода.

Если у вас есть вопрос, как команда Rails управляет определенными API, не стесняйтесь открыть тикет или послать патч в [трекер проблем](https://github.com/rails/rails/issues).
