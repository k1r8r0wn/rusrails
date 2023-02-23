Миграции Active Record
======================

Миграции - это особенность Active Record, позволяющая [изменять схему вашей базы данных время от времени](https://en.wikipedia.org/wiki/Schema_migration). Вместо того, чтобы записывать модификации схемы на чистом SQL, миграции позволяют использовать Ruby DSL для описания изменений в ваших таблицах.

После прочтения этого руководства, вы узнаете о:

* Генераторах, используемых для их создания
* Методах Active Record, обеспечивающих взаимодействие с вашей базой данных
* Задачи rails, воздействующие на миграции и вашу схему
* Как миграции связаны со `schema.rb`

Обзор миграций
--------------

Миграции - это удобный способ изменять схему вашей базы данных всё время последовательным образом. Они используют Ruby DSL. Поэтому вам не нужно писать SQL вручную, позволяя вашей схеме быть независимой от базы данных.

Каждую миграцию можно рассматривать как новую 'версию' базы данных. Схема изначально ничего не содержит, а каждая миграция модифицирует ее, добавляя или убирая таблицы, столбцы или записи. Active Record знает, как обновлять вашу схему со временем, перенося ее из определенной точки в прошлом в последнюю версию. Active Record также обновляет ваш файл `db/schema.rb`, чтобы он соответствовал текущей структуре вашей базы данных.

Вот пример миграции:

```ruby
class CreateProducts < ActiveRecord::Migration[7.1]
  def change
    create_table :products do |t|
      t.string :name
      t.text :description

      t.timestamps
    end
  end
end
```

Эта миграция добавляет таблицу `products` со строковым столбцом `name` и текстовым столбцом `description`. Первичный ключ, названный `id`, также будет неявно добавлен по умолчанию, так как это первичный ключ по умолчанию для всех моделей Active Record. Макрос `timestamps` добавляет два столбца, `created_at` и `updated_at`. Эти специальные столбцы автоматически управляются Active Record, если существуют.

Отметьте, что мы определили изменение, которое мы хотим, чтобы произошло при движении вперед во времени. До запуска этой миграции таблицы нет. После - таблица будет существовать. Active Record также знает, как обратить эту миграцию: если мы откатываем эту миграцию, он удалит таблицу.

В базах данных, поддерживающих транзакции с выражениями, изменяющими схему, миграции оборачиваются в транзакцию. Если база данных это не поддерживает, и миграция проваливается, части, которые прошли успешно, не будут откачены назад. Вам нужно произвести откат вручную.

NOTE: Некоторые запросы не могут быть запущены в транзакции. Если ваш адаптер поддерживает транзакции DDL, можно использовать `disable_ddl_transaction!` для их отключения для отдельной миграции.

Если хотите миграцию для чего-то, что Active Record не знает, как обратить, вы можете использовать `reversible`:

```ruby
class ChangeProductsPrice < ActiveRecord::Migration[7.1]
  def change
    reversible do |dir|
      change_table :products do |t|
        dir.up   { t.change :price, :string }
        dir.down { t.change :price, :integer }
      end
    end
  end
end
```

С другой стороны, можно использовать `up` и `down` вместо `change`:

```ruby
class ChangeProductsPrice < ActiveRecord::Migration[7.1]
  def up
    change_table :products do |t|
      t.change :price, :string
    end
  end

  def down
    change_table :products do |t|
      t.change :price, :integer
    end
  end
end
```

Создание миграции
-----------------

### Создание автономной миграции

Миграции хранятся как файлы в директории `db/migrate`, один файл на каждый класс. Имя файла имеет вид `YYYYMMDDHHMMSS_create_products.rb`, это означает, что временная метка UTC идентифицирует миграцию, затем идет знак подчеркивания, затем идет имя миграции, где слова разделены подчеркиваниями. Имя класса миграции содержит буквенную часть названия файла, но уже в формате CamelCase (т.е. слова пишутся слитно, каждое слово начинается с большой буквы). Например, `20080906120000_create_products.rb` должен определять класс `CreateProducts`, а `20080906120001_add_details_to_products.rb` должен определять `AddDetailsToProducts`. Rails использует эту метку, чтобы определить, какая миграция должна быть запущена и в каком порядке, так что если вы копируете миграции из другого приложения или генерируете файл сами, будьте более бдительны.

Конечно, вычисление временных меток не забавно, поэтому Active Record предоставляет генератор для управления этим:

```bash
$ bin/rails generate migration AddPartNumberToProducts
```

Это создаст правильно названную пустую миграцию:

```ruby
class AddPartNumberToProducts < ActiveRecord::Migration[7.1]
  def change
  end
end
```

Этот генератор может гораздо больше, чем приставить временную метку к имени файла. Основываясь на соглашениях по именованию и дополнительных (необязательных) аргументах, он может также заполнить миграцию.

Если имя миграции имеет форму "AddColumnToTable" или "RemoveColumnFromTable" и далее следует перечень имен столбцов и их типов, то в миграции будут созданы соответствующие выражения [`add_column`][] и [`remove_column`][].

```bash
$ bin/rails generate migration AddPartNumberToProducts part_number:string
```

сгенерирует

```ruby
class AddPartNumberToProducts < ActiveRecord::Migration[7.1]
  def change
    add_column :products, :part_number, :string
  end
end
```

Если вы хотите добавить индекс на новый столбец, это также можно сделать.

```bash
$ bin/rails generate migration AddPartNumberToProducts part_number:string:index
```

сгенерирует необходимые выражения `add_column` and [`add_index`][]:

```ruby
class AddPartNumberToProducts < ActiveRecord::Migration[7.1]
  def change
    add_column :products, :part_number, :string
    add_index :products, :part_number
  end
end
```

Точно так же можно сгенерировать миграцию для удаления столбца из командной строки:

```bash
$ bin/rails generate migration RemovePartNumberFromProducts part_number:string
```

генерирует

```ruby
class RemovePartNumberFromProducts < ActiveRecord::Migration[7.1]
  def change
    remove_column :products, :part_number, :string
  end
end
```

Вы не ограничены одним сгенерированным столбцом. Например:

```bash
$ bin/rails generate migration AddDetailsToProducts part_number:string price:decimal
```

генерирует

```ruby
class AddDetailsToProducts < ActiveRecord::Migration[7.1]
  def change
    add_column :products, :part_number, :string
    add_column :products, :price, :decimal
  end
end
```

Если имя миграции имеет форму "CreateXXX" и затем следует список имен и типов столбцов, то будет сгенерирована миграция, генерирующая таблицу XXX с перечисленными столбцами. Например:

```bash
$ bin/rails generate migration CreateProducts name:string part_number:string
```

генерирует

```ruby
class CreateProducts < ActiveRecord::Migration[7.1]
  def change
    create_table :products do |t|
      t.string :name
      t.string :part_number

      t.timestamps
    end
  end
end
```

Как всегда, то, что было сгенерировано, является всего лишь стартовой точкой. Вы можете добавлять и убирать строки, как считаете нужным, отредактировав файл `db/migrate/YYYYMMDDHHMMSS_add_details_to_products.rb`.

Также генератор принимает такой тип столбца, как `references` (или его псевдоним `belongs_to`). Например,

```bash
$ bin/rails generate migration AddUserRefToProducts user:references
```

генерирует следующий вызов [`add_reference`][]:

```ruby
class AddUserRefToProducts < ActiveRecord::Migration[7.1]
  def change
    add_reference :products, :user, foreign_key: true
  end
end
```

Эта миграция создаст столбец `user_id`. [Ссылки (references)](#references) — это сокращение для создания столбцов, индексов, внешних ключей или даже полимерных столбцов связи.

Существует также генератор, который будет производить объединение таблиц, если `JoinTable` является частью названия.

Например

```bash
$ bin/rails generate migration CreateJoinTableCustomerProduct customer product
```

Сгенерирует следующую миграцию:

```ruby
class CreateJoinTableCustomerProduct < ActiveRecord::Migration[7.1]
  def change
    create_join_table :customers, :products do |t|
      # t.index [:customer_id, :product_id]
      # t.index [:product_id, :customer_id]
    end
  end
end
```

[`add_column`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_column
[`add_index`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_index
[`add_reference`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_reference
[`remove_column`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-remove_column

### Генераторы модели

Генераторы модели, ресурса и скаффолда создадут миграции, подходящие для создания новой модели. Миграция будет содержать инструкции для создания соответствующей таблицы. Если вы сообщите Rails, какие столбцы вы хотите, то выражения для добавления этих столбцов также будут созданы. Например, запуск:

```bash
$ bin/rails generate model Product name:string description:text
```

создаст миграцию, которая выглядит так

```ruby
class CreateProducts < ActiveRecord::Migration[7.1]
  def change
    create_table :products do |t|
      t.string :name
      t.text :description

      t.timestamps
    end
  end
end
```

Можно определить сколько угодно пар имя_столбца/тип.

### Передача модификаторов

Некоторые часто используемые [модификаторы типа](#column-modifiers) могут быть переданы непосредственно в командной строке. Они указываются в фигурных скобках и следуют за типом поля:

К примеру, запуск:

```bash
$ bin/rails generate migration AddDetailsToProducts 'price:decimal{5,2}' supplier:references{polymorphic}
```

создаст миграцию, которая выглядит как эта:

```ruby
class AddDetailsToProducts < ActiveRecord::Migration[7.1]
  def change
    add_column :products, :price, :decimal, precision: 5, scale: 2
    add_reference :products, :supplier, polymorphic: true
  end
end
```

TIP: Чтобы узнать подробности, обратите внимание на выводимые сообщения генератора.

(writing-a-migration) Написание миграции
----------------------------------------

Как только вы создали свою миграцию, используя один из генераторов, пришло время поработать!

### Создание таблицы

Метод [`create_table`][] один из самых фундаментальных, но в большинстве случаев, он будет сгенерирован для вас генератором модели, ресурса или скаффолда. Обычное использование такое

```ruby
create_table :products do |t|
  t.string :name
end
```

Это создаст таблицу `products` со столбцом `name`.

По умолчанию `create_table` создаст первичный ключ, названный `id`. Вы можете изменить имя первичного ключа с помощью опции `:primary_key`, или, если вы вообще не хотите первичный ключ, можно указать опцию `id: false`. Если нужно передать базе данных специфичные опции, вы можете поместить фрагмент `SQL` в опцию `:options`. Например:

```ruby
create_table :products, options: "ENGINE=BLACKHOLE" do |t|
  t.string :name, null: false
end
```

добавит `ENGINE=BLACKHOLE` к SQL выражению, используемому для создания таблицы.

Можно создать индекс на созданные столбцы внутри блока `create_table`, передав true или хэш опций в опцию `:index`:

```ruby
create_table :users do |t|
  t.string :name, index: true
  t.string :email, index: { unique: true, name: 'unique_emails' }
end
```

Также можно передать опцию `:comment` с любым описанием для таблицы, которое будет сохранено в самой базе данных, и может быть просмотрено с помощью инструментов администрирования базы данных, таких как MySQL Workbench или PgAdmin III. Очень рекомендуется оставлять комментарии в миграциях для приложений с большими базами данных, так как это помогает понять модель данных и сгенерировать документацию. В настоящее время комментарии поддерживают только адаптеры MySQL и PostgreSQL.

[`create_table`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-create_table

### Создание соединительной таблицы

Метод миграции [`create_join_table`][] создает соединительную таблицу HABTM (has and belongs to many, многие ко многим). Обычное использование будет таким:

```ruby
create_join_table :products, :categories
```

что создаст таблицу `categories_products` с двумя столбцами по имени `category_id` и `product_id`. У этих столбцов есть опция `:null`, установленная в `false` по умолчанию. Это может быть переопределено опцией `:column_options`:

```ruby
create_join_table :products, :categories, column_options: { null: true }
```

По умолчанию, имя соединительной таблицы получается как соединение первых двух аргументов, переданных в create_join_table, в алфавитном порядке. Чтобы настроить имя таблицы, передайте опцию `:table_name`:

```ruby
create_join_table :products, :categories, table_name: :categorization
```

создает таблицу `categorization`.

По умолчанию `create_join_table` создаст два столбца без опций, но можно определить эти опции с использованием опции `:column_options`. Например,

```ruby
create_join_table :products, :categories, column_options: { null: true }
```

создаст `product_id` и `category_id` с опцией `:null` равной `true`.

`create_join_table` также принимает блок, который можно использовать для добавления индексов (которые по умолчанию не создаются) или дополнительных столбцов:

```ruby
create_join_table :products, :categories do |t|
  t.index :product_id
  t.index :category_id
end
```

[`create_join_table`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-create_join_table

### Изменение таблиц

Близкий родственник `create_table` это [`change_table`][], используемый для изменения существующих таблиц. Он используется подобно `create_table`, но у объекта, передаваемого в блок, больше методов. Например:

```ruby
change_table :products do |t|
  t.remove :description, :name
  t.string :part_number
  t.index :part_number
  t.rename :upccode, :upc_code
end
```

удаляет столбцы `description` и `name`, создает строковый столбец `part_number` и добавляет индекс на него. Наконец, он переименовывает столбец `upccode`.

[`change_table`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-change_table

### Изменение столбцов

Подобно `remove_column` и `add_column`, Rails предоставляет метод миграции [`change_column`][].

```ruby
change_column :products, :part_number, :text
```

Он меняет тип столбца `part_number` в таблице `products` на `:text`. Отметьте, что команда `change_column` — необратима.

Кроме `change_column`, методы [`change_column_null`][] и [`change_column_default`][] используются чтобы изменить ограничение не-null или значение столбца по умолчанию.

```ruby
change_column_null :products, :name, false
change_column_default :products, :approved, from: true, to: false
```

Это настроит поле `:name` в products быть `NOT NULL` столбцом и изменит значение по умолчанию для поля `:approved` с true на false.

NOTE: Также можно написать предыдущую миграцию `change_column_default` как `change_column_default :products, :approved, false`, но, в отличие от предыдущего примера, это сделало бы вашу миграцию необратимой.

[`change_column`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-change_column
[`change_column_default`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-change_column_default
[`change_column_null`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-change_column_null

### (Column Modifiers) Модификаторы столбца

Модификаторы столбца могут быть применены при создании или изменении столбца:

* `comment`      Добавляет комментарий для столбца.
* `collation`    Указывает сопоставление для столбца `string` или `text`.
* `default`      Позволяет установить значение по умолчанию для столбца. Отметьте, что если вы используете динамическое значение (такое как дату), значение по умолчанию будет вычислено лишь один раз (т.е. на дату, когда миграция будет применена). Используйте `nil` для `NULL`.
* `limit`        Устанавливает максимальное количество символов для столбца `string` и максимальное количество байт для столбцов `text/binary/integer`.
* `null`         Позволяет или запрещает значения `NULL` в столбце.
* `precision`    Определяет точность для столбцов `decimal/numeric/datetime/time`.
* `scale`        Определяет масштаб для столбцов `decimal` и `numeric`, определяющий количество цифр после запятой.

NOTE: Для `add_column` или `change_column` нет опций для добавления индексов. Их нужно добавить отдельно с помощью `add_index`.

Некоторые адаптеры могут поддерживать дополнительные опции; за подробностями обратитесь к документации API конкретных адаптеров.

NOTE: С помощью командной строки нельзя указать `null` и `default`.

### (References) Ссылки

Метод `add_reference` позволяет создавать правильно названный столбец.

```ruby
add_reference :users, :role
```

Эта миграция создаст столбец `role_id` в таблице users. Он также создаст индекс для этого столбца, если не задана явно опция `index: false`:

```ruby
add_reference :users, :role, index: false
```

Метод `add_belongs_to` это псевдоним `add_reference`.

```ruby
add_belongs_to :taggings, :taggable, polymorphic: true
```

Опция polymorphic создаст два столбца в таблице taggings. которые можно использовать для полиморфных связей: `taggable_type` и `taggable_id`.

Внешний ключ можно создать с помощью опции `foreign_key`.

```ruby
add_reference :users, :role, foreign_key: true
```

Больше опций `add_reference` в [документации API](https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_reference).

Ссылки также можно убрать:

```ruby
remove_reference :products, :user, foreign_key: true, index: false
```


### (foreign-keys) Внешние ключи

Хотя это и не требуется, вы можете захотеть добавить ограничения внешнего ключа для [обеспечения ссылочной целостности](#active-record-and-referential-integrity).

```ruby
add_foreign_key :articles, :authors
```

Этот вызов [`add_foreign_key`][] добавляет новое ограничение в таблицу `articles`. Это ограничение гарантирует, что существует строка в таблице `authors`, в которой столбец `id` соответствует `articles.author_id`.

Если имя столбца `from_table` не может быть произведен из имени `to_table`, можно использовать опцию `:column`. Используйте опцию `:primary_key`, если ссылаемый первичный ключ не `:id`.

Например, чтобы добавить внешний ключ на `articles.reviewer`, ссылающийся на `authors.email`:

```ruby
add_foreign_key :articles, :authors, column: :reviewer, primary_key: :email
```

`add_foreign_key` также поддерживает такие опции, как `name`, `on_delete`, `if_not_exists`, `validate` и `deferrable`.

NOTE: Active Record поддерживает внешние ключи только для отдельных столбцов. Чтобы использовать составные внешние ключи, требуются `execute` и `structure.sql`. Смотрите [Выгрузка схемы](#schema-dumping-and-you)

Внешний ключ также можно убрать:

```ruby
# позволим Active Record выяснить имя столбца
remove_foreign_key :accounts, :branches

# уберем внешний ключ для определенного столбца
remove_foreign_key :accounts, column: :owner_id
```

### Когда хелперов недостаточно

Если хелперов, предоставленных Active Record, недостаточно, можно использовать метод [`execute`][] для выполнения произвольного SQL:

```ruby
Product.connection.execute("UPDATE products SET price = 'free' WHERE 1=1")
```

Больше подробностей и примеров отдельных методов содержится в документации по API. В частности, документация для [`ActiveRecord::ConnectionAdapters::SchemaStatements`](https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html) (который обеспечивает методы, доступные в методах `up`, `down` и `change`), [`ActiveRecord::ConnectionAdapters::TableDefinition`](https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/TableDefinition.html) (который обеспечивает методы, доступные у объекта, переданного методом `create_table`) и [`ActiveRecord::ConnectionAdapters::Table`](https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/Table.html) (который обеспечивает методы, доступные у объекта, переданного методом `change_table`).

[`execute`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/DatabaseStatements.html#method-i-execute

### Использование метода `change`

Метод `change` это основной метод написания миграций. Он работает в большинстве случаев, в которых Active Record знает, как обратить действия миграции автоматически.  Ниже некоторые действия, которые поддерживает `change`:

* [`add_check_constraint`][]
* [`add_column`][]
* [`add_foreign_key`][]
* [`add_index`][]
* [`add_reference`][]
* [`add_timestamps`][]
* [`change_column_comment`][] (необходимо предоставить опции `:from` и `:to`)
* [`change_column_default`][] (необходимо предоставить опции `:from` и `:to`)
* [`change_column_null`][]
* [`change_table_comment`][] (необходимо предоставить опции `:from` и `:to`)
* [`create_join_table`][]
* [`create_table`][]
* `disable_extension`
* [`drop_join_table`][]
* [`drop_table`][] (необходимо предоставить блок)
* `enable_extension`
* [`remove_check_constraint`][] (необходимо предоставить выражение ограничения)
* [`remove_column`][] (необходимо предоставить тип)
* [`remove_columns`][] (необходимо предоставить опцию `:type`)
* [`remove_foreign_key`][] (необходимо предоставить вторую таблицу)
* [`remove_index`][]
* [`remove_reference`][]
* [`remove_timestamps`][]
* [`rename_column`][]
* [`rename_index`][]
* [`rename_table`][]

[`change_table`][] также является обратимым, когда блок вызывает только обратимые операции, подобные перечисленным.

`remove_column` обратима, если предоставить тип столбца третьим аргументом. Также предоставьте опции оригинального столбца, иначе Rails не сможет в точности пересоздать этот столбец при откате:

```ruby
remove_column :posts, :slug, :string, null: false, default: ''
```

Если вы нуждаетесь в использовании иных методов, следует использовать `reversible` или писать методы `up` и `down` вместо метода `change`.

[`add_check_constraint`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_check_constraint
[`add_foreign_key`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_foreign_key
[`add_timestamps`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-add_timestamps
[`change_column_comment`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-change_column_comment
[`change_table_comment`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-change_table_comment
[`drop_join_table`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-drop_join_table
[`drop_table`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-drop_table
[`remove_check_constraint`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-remove_check_constraint
[`remove_foreign_key`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-remove_foreign_key
[`remove_index`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-remove_index
[`remove_reference`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-remove_reference
[`remove_timestamps`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-remove_timestamps
[`rename_column`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-rename_column
[`remove_columns`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-remove_columns
[`rename_index`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-rename_index
[`rename_table`]: https://api.rubyonrails.org/classes/ActiveRecord/ConnectionAdapters/SchemaStatements.html#method-i-rename_table

### (using-reversible) Использование `reversible`

Комплексная миграция может включать процессы, которые Active Record не знает как обратить. Вы можете использовать [`reversible`][], чтобы указать что делать когда запускается миграция и когда она требует отката. Например:

```ruby
class ExampleMigration < ActiveRecord::Migration[7.1]
  def change
    create_table :distributors do |t|
      t.string :zipcode
    end

    reversible do |dir|
      dir.up do
        # добавим ограничение CHECK
        execute <<-SQL
          ALTER TABLE distributors
            ADD CONSTRAINT zipchk
              CHECK (char_length(zipcode) = 5) NO INHERIT;
        SQL
      end
      dir.down do
        execute <<-SQL
          ALTER TABLE distributors
            DROP CONSTRAINT zipchk
        SQL
      end
    end

    add_column :users, :home_page_url, :string
    rename_column :users, :email, :email_address
  end
end
```

Использование `reversible` гарантирует, что инструкции выполнятся в правильном порядке. Если предыдущий пример миграции откатывается, `down` блок начнёт выполнятся после того как столбец `home_page_url` будет удалён и перед тем как произойдёт удаление таблицы `distributors`.

Иногда миграция будет делать то, что просто необратимо; например, она может уничтожить некоторые данные. В таких случаях, вы можете вызвать `ActiveRecord::IrreversibleMigration` в вашем `down` блоке. Если кто-либо попытается отменить вашу миграцию, будет отображена ошибка, что это не может быть выполнено.

[`reversible`]: https://api.rubyonrails.org/classes/ActiveRecord/Migration.html#method-i-reversible

### Использование методов `up`/`down`

Вы также можете использовать старый стиль миграций используя `up` и `down` методы, вместо `change`.
Метод `up` должен описывать изменения, которые необходимо внести в схему, а метод `down` миграции должен обращать изменения, внесенные методом `up`. Другими словами, схема базы данных должна остаться неизменной после выполнения `up`, а затем `down`. Например, если создать таблицу в методе `up`, ее следует удалить в методе `down`. Разумно производить отмену изменений в полностью противоположном порядке тому, в котором они сделаны в методе `up`. Тогда пример из раздела про `reversible` будет эквивалентен:

```ruby
class ExampleMigration < ActiveRecord::Migration[7.1]
  def up
    create_table :distributors do |t|
      t.string :zipcode
    end

    #добавляем ограничение CHECK
    execute <<-SQL
      ALTER TABLE distributors
        ADD CONSTRAINT zipchk
        CHECK (char_length(zipcode) = 5);
    SQL

    add_column :users, :home_page_url, :string
    rename_column :users, :email, :email_address
  end

  def down
    rename_column :users, :email_address, :email
    remove_column :users, :home_page_url

    execute <<-SQL
      ALTER TABLE distributors
        DROP CONSTRAINT zipchk
    SQL

    drop_table :distributors
  end
end
```

Если ваша миграция необратима вам следует вызвать `ActiveRecord::IrreversibleMigration` из вашего метода `down`. Если кто-либо попытается отменить вашу миграцию, будет отображена ошибка, что это не может быть выполнено.

### (reverting-previous-migrations) Возвращение к предыдущим миграциям

Вы можете использовать возможность Active Record, чтобы откатить миграции с помощью метода [`revert`][]:

```ruby
require_relative "20121212123456_example_migration"

class FixupExampleMigration < ActiveRecord::Migration[7.1]
  def change
    revert ExampleMigration

    create_table(:apples) do |t|
      t.string :variety
    end
  end
end
```

Метод `revert` также может принимать блок. Это может быть полезно для отката выбранной части предыдущих миграций. Для примера, давайте представим, что `ExampleMigration` закоммичена, а позже мы решили, что было бы лучше использовать валидации Active Record, вместо ограничения `CHECK`, для проверки zipcode.

```ruby
class DontUseConstraintForZipcodeValidationMigration < ActiveRecord::Migration[7.1]
  def change
    revert do
      reversible do |dir|
        dir.up do
          # добавим ограничение CHECK
          execute <<-SQL
            ALTER TABLE distributors
              ADD CONSTRAINT zipchk
                CHECK (char_length(zipcode) = 5);
          SQL
        end
        dir.down do
          execute <<-SQL
            ALTER TABLE distributors
              DROP CONSTRAINT zipchk
          SQL
        end
      end

      # The rest of the migration was ok
    end
  end
end
```

Подобная миграция также может быть написана без использования `revert`, но это бы привело к ещё нескольким шагам: изменение порядка (следования) `create table` и `reversible`, замена `create_table` на `drop_table` и в конечном итоге изменение `up` на `down` и наоборот. Обо всём этом уже позаботился `revert`.

[`revert`]: https://api.rubyonrails.org/classes/ActiveRecord/Migration.html#method-i-revert

Запуск миграций
---------------

Rails предоставляет ряд команд rails для запуска определенных наборов миграций.

Самая первая миграция, относящаяся к команде rails, которую будем использовать, это `bin/rails db:migrate`. В своей основной форме она всего лишь запускает метод `change` или `up` для всех миграций, которые еще не были запущены. Если таких миграций нет, она выходит. Она запустит эти миграции в порядке, основанном на дате миграции.

Заметьте, что запуск команды `db:migrate` также вызывает команду `db:schema:dump`, которая обновляет ваш файл `db/schema.rb` в соответствии со структурой вашей базы данных.

Если вы определите целевую версию, Active Record запустит требуемые миграции (методы up, down или change), пока не достигнет требуемой версии. Версия это числовой префикс у файла миграции. Например, чтобы мигрировать к версии 20080906120000, запустите:

```bash
$ bin/rails db:migrate VERSION=20080906120000
```

Если версия 20080906120000 больше текущей версии (т.е. миграция вперед) это запустит метод `change` (или `up`) для всех миграций до и включая 20080906120000, но не выполнит какие-либо поздние миграции. Если миграция назад, это запустит метод `down` для всех миграций до, но не включая, 20080906120000.

### Откат

Обычная задача - это откатить последнюю миграцию. Например, вы сделали ошибку и хотите исправить ее. Можно отследить версию предыдущей миграции и произвести миграцию до нее, но можно поступить проще, запустив:

```bash
$ bin/rails db:rollback
```

Это вернёт ситуацию к последней миграции, или обратив метод `change`, или запустив метод `down`. Если нужно отменить несколько миграций, можно указать параметр `STEP`:

```bash
$ bin/rails db:rollback STEP=3
```

произойдёт откат на 3 последних миграции.

Команда `db:migrate:redo` это ярлык для выполнения отката, а затем запуска миграции снова. Так же, как и с командой `db:rollback` можно указать параметр `STEP`, если нужно работать более чем с одной версией, например:

```bash
$ bin/rails db:migrate:redo STEP=3
```

Ни одна из этих команд rails не может сделать ничего такого, чего нельзя было бы сделать с `db:migrate`. Они там для удобства, так как вам не нужно явно указывать версию миграции, к которой нужно мигрировать.

### Настройка базы данных

Команда `bin/rails db:setup` создаст базу данных, загрузит схему и инициализирует ее с помощью данных seed.

### Сброс базы данных

Команда `bin/rails db:reset` удалит базу данных и установит ее заново. Функционально это эквивалентно `bin/rails db:drop db:setup`.

NOTE. Это не то же самое, что запуск всех миграций. Будет использовано только текущее содержимое файла `db/schema.rb` или `db/structure.sql`. Если миграцию откатить невозможно, `bin/rails db:reset` может не помочь вам. Подробнее о выгрузке схемы смотрите раздел [Выгрузка схемы](#schema-dumping-and-you).

### Запуск определенных миграций

Если необходимо запустить определённую миграцию вверх или вниз, это делают команды `db:migrate:up` и `db:migrate:down`. Просто укажите подходящую версию и у соответствующей миграции будет вызван метод `change`, `up` или `down`, например:

```bash
$ bin/rails db:migrate:up VERSION=20080906120000
```

запустит метод `up` у миграции 20080906120000. Эта команда сперва проверит, была ли миграция уже выполнена, и ничего делать не будет, если Active Record считает, что она уже была запущена.

### Запуск миграций в различных средах

По умолчанию запуск `bin/rails db:migrate` запустится в окружении `development`. Для запуска миграций в другом окружении, его можно указать, используя переменную среды `RAILS_ENV` при запуске команды. Например, для запуска миграций в среде `test`, следует запустить:

```bash
$ bin/rails db:migrate RAILS_ENV=test
```

### Изменение вывода результата запущенных миграций

По умолчанию миграции говорят нам только то, что они делают, и сколько времени это заняло. Миграция, создающая таблицу и добавляющая индекс, выдаст что-то наподобие этого:

```
==  CreateProducts: migrating =================================================
-- create_table(:products)
   -> 0.0028s
==  CreateProducts: migrated (0.0028s) ========================================
```

Некоторые методы в миграциях позволяют вам все это контролировать:

| Метод                   | Назначение
| ----------------------- |-----------
| [`suppress_messages`][] | Принимает блок как аргумент и запрещает любой вывод, сгенерированный этим блоком.
| [`say`][]               | Принимает сообщение как аргумент и выводит его как есть. Может быть передан второй булевый аргумент для указания, нужен отступ или нет.
| [`say_with_time`][]     | Выводит текст вместе с продолжительностью выполнения блока. Если блок возвращает число, предполагается, что это количество затронутых строк.

Например, эта миграция:

```ruby
class CreateProducts < ActiveRecord::Migration[7.1]
  def change
    suppress_messages do
      create_table :products do |t|
        t.string :name
        t.text :description
        t.timestamps
      end
    end

    say "Created a table"

    suppress_messages {add_index :products, :name}
    say "and an index!", true

    say_with_time 'Waiting for a while' do
      sleep 10
      250
    end
  end
end
```

сгенерирует следующий результат

```
==  CreateProducts: migrating =================================================
-- Created a table
   -> and an index!
-- Waiting for a while
   -> 10.0013s
   -> 250 rows
==  CreateProducts: migrated (10.0054s) =======================================
```

Если хотите, чтобы Active Record ничего не выводил, запуск `bin/rails db:migrate VERBOSE=false` запретит любой вывод.

[`say`]: https://api.rubyonrails.org/classes/ActiveRecord/Migration.html#method-i-say
[`say_with_time`]: https://api.rubyonrails.org/classes/ActiveRecord/Migration.html#method-i-say_with_time
[`suppress_messages`]: https://api.rubyonrails.org/classes/ActiveRecord/Migration.html#method-i-suppress_messages

Изменение существующих миграций
-------------------------------

Периодически вы будете делать ошибки при написании миграции. Если вы уже запустили миграцию, вы не сможете просто отредактировать миграцию и запустить ее снова: Rails посчитает, что он уже выполнял миграцию, и ничего не сделает при запуске `bin/rails db:migrate`. Вы должны откатить миграцию (например, с помощью `bin/rails db:rollback`), отредактировать миграцию и затем запустить `bin/rails db:migrate` для запуска исправленной версии.

В целом, редактирование существующих миграций не хорошая идея. Вы создадите дополнительную работу себе и своим коллегам, и вызовете море головной боли, если существующая версия миграции уже была запущена в production. Вместо этого, следует написать новую миграцию, выполняющую требуемые изменения. Редактирование только что сгенерированные миграции, которая еще не была закоммичена в систему контроля версий (или, хотя бы, не ушла дальше вашей рабочей машины) относительно безвредно.

Метод `revert` может быть очень полезным при написании новой миграции для возвращения предыдущей миграции в целом или какой-то ее части (смотрите [Возвращение к предыдущим миграциям](#reverting-previous-migrations)).

(Schema Dumping and You) Выгрузка схемы
---------------------------------------

### Для чего нужны файлы схемы?

Миграции, какими бы не были они мощными, не являются авторитетным источником для схемы базы данных. База данных остается авторитетным источником. По умолчанию Rails генерирует `db/schema.rb`, которая пытается охватить текущее состояние схемы базы данных.

Она имеет тенденцию быть более быстрой и менее подверженной ошибкам, связанным с созданием нового экземпляра базы данных приложения, загружая файл схемы через `bin/rails db:schema:load`, чем при повторном воспроизведении всей истории миграций. [Старые миграции](#old-migrations) могут работать неправильно, если эти миграции используют изменения внешних зависимостей или полагаются на код приложения, который развивается отдельно от этих миграций.

Файлы схемы также полезны, если необходимо быстро посмотреть, какие атрибуты есть у объекта Active Record. Эта информация не содержится в коде модели и часто распределена по нескольким миграциям, но собрана воедино в файле схемы.

### Типы выгрузок схемы

Формат выгрузки схемы, сгенерированный Rails, управляется настройкой [`config.active_record.schema_format`][] в `config/application.rb`. Форматом по умолчанию является `:ruby`, но также он может быть установлен в `:sql`.

Если выбрано `:ruby`, тогда схема хранится в `db/schema.rb`. Посмотрев в этот файл, можно увидеть, что он очень похож на одну большую миграцию:

```ruby
ActiveRecord::Schema[7.1].define(version: 2008_09_06_171750) do
  create_table "authors", force: true do |t|
    t.string   "name"
    t.datetime "created_at"
    t.datetime "updated_at"
  end

  create_table "products", force: true do |t|
    t.string   "name"
    t.text     "description"
    t.datetime "created_at"
    t.datetime "updated_at"
    t.string   "part_number"
  end
end
```

Во многих случаях этого достаточно. Этот файл создается путем проверки базы данных и описывает свою структуру, используя `create_table`, `add_index` и так далее.

`db/schema.rb` не может описать все, что может поддерживать база данных, например триггеры, последовательности, хранимые процедуры и так далее. Отметьте, в то время как в миграциях можно выполнить произвольные выражения SQL, эти выражения не смогут быть воспроизведены выгрузчиком схемы. Если используете подобные особенности, необходимо установить формат схемы как `:sql`, чтобы получить точный файл схемы, который будет полезен для создания новых экземпляров базы данных.

Когда формат схемы установлен в `:sql`, структура базы данных будет выгружена с помощью инструмента, предназначенного для этой базы данных в `db/structure.sql`. Например, для PostgreSQL используется утилита `pg_dump`. Для MySQL и MariaDB этот файл будет содержать результат `SHOW CREATE TABLE` для разных таблиц.

Чтобы загрузить схему из `db/structure.sql`, запустите `bin/rails db:structure:load`. Загрузка этого файла осуществляется путем выполнения содержащихся в нем выражений SQL. По определению создастся точная копия структуры базы данных.

[`config.active_record.schema_format`]: /configuring#config-active-record-schema-format

### Выгрузки схем и управление версиями

Поскольку файлы схемы обычно используются для создания новых баз данных, настоятельно рекомендуется проверять файл схемы в системе управления версиями.

Конфликты слияния могут возникать в файле схемы, когда две ветки модифицируют схему. Для разрешения этих конфликтов, запустите `bin/rails db:migrate`, чтобы восстановить файл схемы.

(Active Record and Referential Integrity) Active Record и ссылочная целостность
-------------------------------------------------------------------------------

Способ Active Record требует, чтобы логика была в моделях, а не в базе данных. По большому счету, функции, такие как триггеры или ограничения, которые переносят часть логики обратно в базу данных, не используются активно.

Валидации, такие как `validates :foreign_key, uniqueness: true`, это один из способов, которым ваши модели могут соблюдать ссылочную целостность. Опция `:dependent` в связях позволяет моделям автоматически уничтожать дочерние объекты при уничтожении родителя. Подобно всему, что работает на уровне приложения, это не может гарантировать ссылочной целостности, таким образом кто-то может добавить еще и [внешние ключи как ограничители ссылочной целостности](#foreign-keys) в базе данных.

Хотя Active Record не предоставляет каких-либо инструментов для работы напрямую с этими функциями, метод `execute` может использоваться для выполнения произвольного SQL.

Миграции и сиды
---------------

Основным назначением миграции Rails является запуск команд, последовательно модифицирующих схему. Миграции также могут быть использованы для добавления или модифицирования данных. Это полезно для существующей базы данных, которую нельзя удалить и пересоздать, такой как база данных на production.

```ruby
class AddInitialProducts < ActiveRecord::Migration[7.1]
  def up
    5.times do |i|
      Product.create(name: "Product ##{i}", description: "A product.")
    end
  end

  def down
    Product.delete_all
  end
end
```

Чтобы добавить изначальные данные в базу данных после создания, в Rails имеется встроенная особенность 'seeds', которая ускоряет процесс. Это особенно полезно при частой перезагрузке базы данных в средах разработки и тестирования. Чтобы начать пользоваться этой особенностью, заполните `db/seeds.rb` некоторым кодом Ruby и запустите `bin/rails db:seed`:

```ruby
5.times do |i|
  Product.create(name: "Product ##{i}", description: "A product.")
end
```

В основном, это более чистый способ настроить базу данных для пустого приложения.

(Old Migrations) Старые миграции
--------------------------------

`db/schema.rb` или `db/structure.sql` это снимок текущего состояния вашей базы данных и авторитетный источник для восстановления этой базы данных. Поэтому возможно удалить старые файлы миграций.

Когда вы удалите файлы миграций в директории `db/migrate/`, любая среда, в которой `bin/rails db:migrate` была запущена, когда эти файлы еще существовали, будет хранить ссылки на временные метки миграций во внутренней таблице Rails по имени `schema_migrations`. Эта таблица используется для отслеживания, была ли миграция выполнена в указанной среде.

Если вы запустите команду `bin/rails db:migrate:status`, которая отображает статус
(up или down) каждой миграции, вы увидите `********** NO FILE **********`, отображенный рядом с каждым удаленным файлом миграции, который однажды был запущен в указанной среде, но больше не найден в директории `db/migrate/`.

Хотя тут есть предостережение. Задачи Rake для установки миграций из engine являются идемпотентными. Миграции, присутствующие в родительском приложении благодаря предыдущим установками, пропускаются, а отсутствующие копируются с новой временной меткой. Если вы удалите старые миграции engine и запустите задачу установки заново, вы получите новые файлы с новыми временными метками, и `db:migrate` попытается запустить их снова.

Поэтому, как правило, вам захочется сохранить миграции, пришедшие из engine. В них есть специальный комментарий, наподобие:

```
# This migration comes from blorgh (originally 20210621082949)
```
