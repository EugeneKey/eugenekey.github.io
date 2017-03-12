---
layout: post
title:  "Что нового в Rails 5"
image: ''
date:   2017-02-17 14:17:26
tags:
- rails
description: ''
categories:
- Ruby on Rails
---

## Ключевые новинки в Rails 5.0

- Action Cable
- Rails API
- API атрибутов Active Record
- Test Runner
- Эксклюзивное использование интерфейса командной строки <code>rails</code>вместо Rake
- Sprockets 3
- Turbolinks 5
- Требуется Ruby 2.2.2+

Эти заметки о релизе покрывают только основные обновления. Чтобы узнать о других обновлениях, различных багфиксах и изменениях, обратитесь к логам изменений или к <a href="https://github.com/rails/rails/commits/5-0-stable" target="_blank">списку коммитов</a> в главном репозитории Rails на GitHub.

## 1. Основные изменения

### 1.1. Action Cable

Action Cable — это новый фреймворк в Rails 5. Он с легкостью интегрирует WebSockets с остальными частями вашего приложения Rails.

Action Cable позволяет писать функционал реального времени на Ruby в стиле и формате остальной части приложения Rails, в то же время являясь производительным и масштабируемым. Он представляет полный стек, включая клиентский фреймворк на JavaScript и серверный фреймворк на Ruby. Вы получаете доступ к моделям, написанным с помощью Active Record или другой ORM.

### 1.2. Rails API

дополняется...

### 1.3. API атрибутов Active Record

Определяет в модели тип с атрибутом. Это позволит при необходимости переопределить тип существующих атрибутов. Это позволяет контролировать, как значения конвертируются в и из SQL при присвоении модели. Это также изменяет поведение значений, переданных в <code>ActiveRecord::Base.where</code>, что позволяет использовать наши объекты предметной области в большей части Active Record не полагаясь на особенности реализации или monkey patching.

Некоторые из вещей, которые можно достичь с помощью этого: 
- Тип, распознанный Active Record, может быть переопределен.
- Также может быть представлено значение по умолчанию.
- Атрибутам не обязательно должен соответствовать столбец базы данных.

{% highlight ruby %}

# db/schema.rb
create_table :store_listings, force: true do |t|
  t.decimal :price_in_cents
  t.string :my_string, default: "original default"
end
 
# app/models/store_listing.rb
class StoreListing < ActiveRecord::Base
end
 
store_listing = StoreListing.new(price_in_cents: '10.1')
 
# before
store_listing.price_in_cents # => BigDecimal.new(10.1)
StoreListing.new.my_string # => "original default"
 
class StoreListing < ActiveRecord::Base
  attribute :price_in_cents, :integer # настраиваемый тип
  attribute :my_string, :string, default: "new default" # значение по умаолчанию
  attribute :my_default_proc, :datetime, default: -> { Time.now } # значение по умолчанию
  attribute :field_without_db_column, :integer, array: true
end
 
# after
store_listing.price_in_cents # => 10
StoreListing.new.my_string # => "new default"
StoreListing.new.my_default_proc # => 2015-05-30 11:04:48 -0600
model = StoreListing.new(field_without_db_column: ["1", "2", "3"])
model.attributes #=> {field_without_db_column: [1, 2, 3]}

{% endhighlight %}


**Создание собственных типов:**

Можно определить свои собственные типы, но они должны отвечать на методы для определенного типа значения. Метод +deserialize+ или +cast+ будет вызван на вашем объекте типа с необработанными данными из базы данных или от контроллера. Это полезно, к примеру, при осуществлении пользовательских преобразований, таких как данные Money.

**Запросы:**

При вызове <code>ActiveRecord::Base.where</code>, он будет использовать тип, определенный классом модели, для конвертации значения в SQL, вызвав +serialize+ на вашем объекте типа.

Это дает объектам способность указывать, как конвертировать значения при выполнении запросов SQL.

**Отслеживание изменений (Dirty Tracking):**

Тип атрибута дает возможность изменить способ, как выполняется отслеживание изменений.

Подробности смотрите в его <a href="http://api.rubyonrails.org/classes/ActiveRecord/Attributes/ClassMethods.html" target="_blank">документации</a>.

### 1.4. Test Runner

дополняется...

## 2. Railties

За подробностями обратитесь к <a href="https://github.com/rails/rails/blob/5-0-stable/railties/CHANGELOG.md" target="_blank">Changelog</a>.

### 2.1. Удалено

Удалена поддержка <code>debugger</code>, используйте вместо него <code>byebug</code>. <code>debugger</code> больше не поддерживается Ruby 2.2.

Удалены устаревшие задачи <code>test:all</code> и <code>test:all:db</code>.

Удален устаревший <code>Rails::Rack::LogTailer</code>.

Удалена устаревшая константа <code>RAILS_CACHE</code>.

Удалена устаревшая настройка <code>serve_static_assets</code>.

Удалены задачи для документации <code>doc:app</code>, <code>doc:rails</code> и <code>doc:guides</code>.

Из стека по умолчанию удалена промежуточная программа <code>Rack::ContentLength</code>.

### 2.2. Устарело

Устарела <code>config.static_cache_control</code> в пользу <code>config.public_file_server.headers</code>.

Устарела <code>config.serve_static_files</code> в пользу <code>config.public_file_server.enabled</code>.

Устарели задачи в пространстве имен <code>rails</code> в пользу пространства имен <code>app</code>. (например, задачи <code>rails:update</code> и <code>rails:template</code> переименованы в <code>app:update</code> и <code>app:template</code>.)

### 2.3. Значимые изменения

Добавлен Rails test runner <code>bin/rails test</code>.

Вновь сгенерированные приложения и плагины получают README.md в формате Markdown.

Добавлена задача <code>bin/rails restart</code> для перезапуска вашего приложения Rails, изменяя время <code>tmp/restart.txt</code>.

Добавлена задача <code>bin/rails initializers</code>, выводящая все определенные инициализаторы в том порядке, в котором они вызываются Rails.

Добавлена <code>bin/rails dev:cache</code> для включения или отключения кэширования в режиме разработки.

Добавлен скрипт <code>bin/update</code> для автоматического обновления среды development.

Проксируются задачи Rake с помощью <code>bin/rails</code>.

Новые приложения генерируются с включенным монитором событийной файловой системы на Linux и Mac OS X. Эта особенность может быть отключена, передав <code>--skip-listen</code> в генератор.

Генерация приложений с опцией вывода лога в STDOUT в production с помощью переменной среды <code>RAILS_LOG_TO_STDOUT</code>>.

Для новых приложений включен HSTS с заголовком IncludeSudomains.

Генератор приложения создает новый файл <code>config/spring.rb</code>, который сообщает Spring наблюдать за дополнительными распространенными файлами.

Добавлена <code>--skip-action-mailer</code>, чтобы пропустить Action Mailer при генерации нового приложения.

Убрана директория <code>tmp/sessions</code> и задача очистки rake, связанная с ней.

Изменен <code>_form.html.erb</code>, генерируемый скаффолдом, чтобы использовались локальные переменные.

## 3. Action Pack

За подробностями обратитесь к <a href="https://github.com/rails/rails/blob/5-0-stable/actionpack/CHANGELOG.md" target="_blank">Changelog</a>.

### 3.1. Удалено

Удален <code>ActionDispatch::Request::Utils.deep_munge</code>.

Удален <code>ActionController::HideActions</code>.

Удалены методы <code>respond_to</code> и <code>respond_with</code>>, этот функционал был извлечен в гем <code>responders</code>.

Удалены устаревшие файлы тестовых утверждений.

Удалено устаревшее использование строковых ключей в хелперах путей.

Удалена устаревшая опция <code>only_path</code> в хелперах <code>*_path</code>.

Удален устаревший <code>NamedRouteCollection#helpers</code>.

Удалена устаревшая поддержка определения маршрутов с помощью опции <code>:to</code>, не содержащей <code>#</code>.

Удален устаревший <code>ActionDispatch::Response#to_ary</code>.

Удален устаревший <code>ActionDispatch::Request#deep_munge</code>.

Удален устаревший <code>ActionDispatch::Http::Parameters#symbolized_path_parameters</code>.

Удалена устаревшая опция <code>use_route</code> в тестах контроллеров.

Удалены <code>assigns</code> и <code>assert_template</code>. Оба метода были извлечены в гем <code>rails-controller-testing</code>.

### 3.2. Устарело

Устарели все колбэки <code>*_filter</code> в пользу колбэков <code>*_action</code>.

Устарели интеграционные методы тестирования <code>*_via_redirect</code>. Используйте вручную <code>follow_redirect!</code> после вызова запроса для того же поведения.

Устарел <code>AbstractController#skip_action_callback</code> в пользу отдельных методов <code>skip_callback</code>.

Устарела опция <code>:nothing</code> для метода <code>render</code>.

Устарела передача первого параметра как <code>Hash</code> и код статуса по умолчанию для метода <code>head</code>.

Устарело использование строк или символов для имен классов промежуточных программ. Используйте вместо них имена классов.

Устарел доступ к типам <code>mime</code> с помощью констант (т.е. <code>Mime::HTML</code>). Вместо них используйте оператор индексирования с символом (т.е. <code>Mime[:html]</code>).

Устарел <code>redirect_to :back</code> в пользу <code>redirect_back</code>, который принимает аргумент <code>fallback_location</code>, устраняющий возможность <code>RedirectBackError</code>.

В <code>ActionDispatch::IntegrationTest</code> и <code>ActionController::TestCase</code> устарели позиционные аргументы в пользу аргументов с ключевым словом.

Устарели параметры пути <code>:controller</code> и <code>:action</code>.

### 3.3. Значимые изменения

Добавлен <code>ActionController::Renderer</code> для рендера произвольных шаблонов вне экшнов контроллера.

Произошел переход на синтаксис с ключевыми аргументами в методах запроса HTTP <code>ActionController::TestCase</code> и <code>ActionDispatch::Integration</code>.

В <code>Action Controller</code> добавлен <code>http_cache_forever</code>, таким образом можно кэшировать отклик, который никогда не устаревает.

Предоставлен более дружелюбный доступ к вариантам запроса.

Для экшнов без соответствующих шаблонов рендерится <code>head :no_content</code> вместо вызова ошибки.

Добавлена возможность переопределить билдер формы по умолчанию для контроллера.

Добавлена поддержка для чистых API-приложений. Добавлен <code>ActionController::API</code> в качестве замены <code>ActionController::Base</code> для такого типа приложений.

<code>ActionController::Parameters</code> больше не наследуется от <code>HashWithIndifferentAccess</code>.

Упрощена настройка <code>config.force_ssl</code> и <code>config.ssl_options</code>, они сделаны менее опасными для пробы и более простыми для отключения.

Добавлена возможность возврата произвольных заголовков в <code>ActionDispatch::Static</code>.

Изменено значение по умолчанию для опции prepend метода <code>protect_from_forgery на false</code>.

<code>ActionController::TestCase</code> будет перемещен в отдельный гем в Rails 5.1. Вместо него используйте <code>ActionDispatch::IntegrationTest</code>.

Rails будет генерировать только "слабые", в отличие от сильных ETag.

Экшны контроллера без явного вызова <code>render</code> и без соответствующих шаблонов будут неявно рендерить <code>head :no_content</code> вместо вызова ошибки.

Добавлена опция для CSRF токенов для отдельной формы.

Добавлены кодировка запроса и парсинг отклика в интеграционные тесты.

Обновлены политики рендеринга по умолчанию, когда экшн контроллера не указывает явно отклик.

Добавлен <code>ActionController#helpers</code> для получения доступа к контексту вьюхи на уровне контроллера.

Показанные сообщения <code>flash</code> убираются перед сохранением в сессию.

## 4. Action View
За подробностями обратитесь к <a href="https://github.com/rails/rails/blob/5-0-stable/actionview/CHANGELOG.md" target="_blank">Changelog</a>.

### 4.1. Удалено

Уделен устаревший <code>AbstractController::Base::parent_prefixes</code>.

Удален <code>ActionView::Helpers::RecordTagHelper</code>, этот функционал был извлечен в гем <code>record_tag_helper</code>.

Убрана опция <code>:rescue_format</code> для хелпера <code>translate</code>, так как она больше не поддерживается i18n.

### 4.2. Устарело

Устарели хелперы <code>datetime_field</code> и <code>datetime_field_tag</code>. Тип ввода <code>datetime</code> был убран из спецификации HTML. Вместо них можно использовать <code>datetime_local_field</code> и <code>datetime_local_field_tag</code>.
5.3. Значимые изменения

Изменен обработчик шаблонов по умолчанию с ERB на Raw.

Рендеринг коллекций может кэшировать и извлекать несколько партиалов за раз.

Добавлено универсальное сопоставление для явных зависимостей.

<code>disable_with</code> сделано поведением по умолчанию для тегов submit. Отключает кнопку при отправке, чтобы предотвратить двойную отправку.

Имя шаблона партиала больше не обязано быть валидным идентификатором Ruby.

## 5. Action Mailer
За подробностями обратитесь к <a href="https://github.com/rails/rails/blob/5-0-stable/actionmailer/CHANGELOG.md" target="_blank">Changelog</a>.

### 5.1. Удалено

Удалены устаревшие хелперы <code>*_path</code> во вьюхах email.

Удалены устаревшие методы <code>deliver</code> и <code>deliver!</code>.

### 5.2. Значимые изменения

Поиск шаблонов теперь учитывает локаль по умолчанию и фолбэки I18n.

Рассыльщикам, создаваемым генератором, добавляется суффикс <code>_mailer</code>, в соответствии с соглашениями об именовании, использованными в контроллерах и задачах.

Добавлены <code>assert_enqueued_emails</code> и <code>assert_no_enqueued_emails</code>.

Добавлена настройка <code>config.action_mailer.deliver_later_queue_name</code> для установления имени очереди рассыльщика.

Добавлена поддержка кэширования фрагмента во вьюхах Action Mailer. Добавлена новая конфигурационная опция <code>config.action_mailer.perform_caching</code> для определения, должны ли ваши шаблоны осуществлять кэширование или нет.

## 6. Active Record

За подробностями обратитесь к <a href="https://github.com/rails/rails/blob/5-0-stable/activerecord/CHANGELOG.md" target="_blank">Changelog</a>.

### 6.1. Удалено

Удалено устаревшее поведение, позволяющее передавать вложенные массивы в качестве значений запроса.

Удален устаревший <code>ActiveRecord::Tasks::DatabaseTasks#load_schema</code>. Этот метод был заменен <code>ActiveRecord::Tasks::DatabaseTasks#load_schema_for</code>.

Удален устаревший <code>serialized_attributes</code>.

Удалены устаревшие автоматические кэши счетчиков на <code>has_many :through</code>.

Удален устаревший <code>sanitize_sql_hash_for_conditions</code>.

Удален устаревший <code>Reflection#source_macro</code>.

Удалены устаревшие <code>symbolized_base_class</code> и <code>symbolized_sti_name</code>.

Удалены устаревшие <code>ActiveRecord::Base.disable_implicit_join_references=</code>.

Удален устаревший доступ к спецификации соединения с помощью строкового метода доступа.

Удалена устаревшая поддержка предварительной загрузки связей, зависимых от экземпляра.

Удалена устаревшая поддержка интервалов PostgreSQL с исключенной нижней границей.

Удалено предупреждение об устаревании при при изменении relation с кэшированным Arel. Вместо этого вызывается ошибка <code>ImmutableRelation</code>.

Из ядра удален <code>ActiveRecord::Serialization::XmlSerializer</code>. Эта особенность была извлечена в гем <code>activemodel-serializers-xml</code>.

Из ядра удалена поддержка старой версии адаптера баз данных <code>mysql</code>. Пока что он будет существовать в отдельном геме, но большинство пользователей должны просто использовать <code>mysql2</code>.

Удалена поддержка гема <code>protected_attributes</code>.

Удалена поддержка для PostgreSQL версии ниже 9.1.

Удалена поддержка гема <code>activerecord-deprecated_finders</code>.

### 6.2. Устарело

Устарела передача класса в качестве значения запроса. Вместо этого нужно передавать строки.

Устарел возврат <code>false</code> в качестве способа прервать цепочку колбэков Active Record. Рекомендуемый способ <code>throw(:abort)</code>.

Устарел <code>ActiveRecord::Base.errors_in_transactional_callbacks=</code>.

Устарел <code>Relation#uniq</code>, вместо него используйте Relation#distinct.

Устарел тип PostgreSQL <code>:point</code> в пользу нового, возвращающего объекты Point вместо Array

Устарело принуждение к перезагрузке связи с помощью передачи истинного аргумента в метод связи.

Устарели ключи для ошибок связи <code>restrict_dependent_destroy</code> в пользу новых имен ключей.

Синхронизировано поведение <code>#tables</code>.

Устарели <code>SchemaCache#tables</code>, <code>SchemaCache#table_exists?</code> и <code>SchemaCache#clear_table_cache!</code> в пользу их новых дубликатов <code>data_source</code>.

Устарел <code>connection.tables</code> в адаптерах SQLite3 и MySQL.

Устарела передача аргументов в <code>#tables</code> - метод <code>#tables</code> в некоторых адаптерах (<code>mysql2</code>, <code>sqlite3</code>) мог возвращать и таблицы, и представления, в то время как другие (<code>postgresql</code>) просто возвращали таблицы. Чтобы сделать их поведение согласующимся, в будущем <code>#tables</code> будет возвращать только таблицы.

Устарел <code>table_exists?</code> - метод <code>#table_exists?</code> мог проверять и таблицы, и представления. Чтобы сделать его поведение согласующимся с <code>#tables</code>, в будущем <code>#table_exists?</code> будет проверять только таблицы.

Устарела отправка аргумента <code>offset</code> в <code>find_nth</code>. Вместо этого используйте метод <code>offset</code> на <code>relation</code>.

Устарели <code>{insert|update|delete}_sql</code> в DatabaseStatements. Вместо этого используйте публичные методы <code>{insert|update|delete}</code>.

Устарел <code>use_transactional_fixtures</code> в пользу <code>use_transactional_tests</code> для большей ясности.

### 6.3. Значимые изменения

Добавлена опция <code>foreign_key</code> в <code>references</code> во время создания таблицы.

Новый API атрибутов.

Добавлена опция <code>:_prefix/:_suffix</code> в определении enum.

Добавлен <code>#cache_key</code> в <code>ActiveRecord::Relation</code>.

Изменено значение по умолчанию <code>null</code> для <code>timestamps</code> на <code>false</code>.

Добавлен <code>ActiveRecord::SecureToken</code>, чтобы инкапсулировать генерацию уникальных токенов для атрибутов модели с помощью <code>SecureRandom</code>.

Добавлена опция <code>:if_exists</code> для <code>drop_table</code>.

Добавлен <code>ActiveRecord::Base#accessed_fields</code>, который может быть использован, чтобы быстро просмотреть, какие поля были прочитаны из модели, когда вы выбираете только те данные из базы данных, которые вам нужны.

Добавлен метод <code>#or</code> на <code>ActiveRecord::Relation</code>, позволяющий использование оператора <code>OR</code> в сочетании с выражениями <code>WHERE</code> или <code>HAVING</code>.

Добавлена опция <code>:time</code> для <code>#touch</code>.

Добавлен <code>ActiveRecord::Base.suppress</code> предотвращающий получатель от сохранения в заданном блоке.

<code>belongs_to</code> по умолчанию теперь вызывает ошибку валидации, если связь не существует. Это можно отключить для конкретной связи с помощью <code>optional: true</code>. Также устарела опция <code>required</code> в пользу <code>optional</code> для <code>belongs_to</code>.

Добавлен <code>config.active_record.dump_schemas</code> для настройки поведения <code>db:structure:dump</code>.

Добавлена опция <code>config.active_record.warn_on_records_fetched_greater_than</code>.

Добавлена поддержка нативного типа данных <code>JSON</code> в MySQL.

Добавлена поддержка для параллельного удаления индексов в PostgreSQL.

Добавлены методы <code>#views</code> и <code>#view_exists?</code> на адаптерах соединений.

Добавлен <code>ActiveRecord::Base.ignored_columns</code>, чтобы сделать некоторые столбцы невидимыми из Active Record.

Добавлены <code>connection.data_sources</code> и <code>connection.data_source_exists?</code>. Эти методы определяют, какие <code>relation</code> могут быть использованы для создание моделей Active Record (обычно таблицы и представления).

В файлах фикстур можно указать класс модели в самом файле <code>YAML</code>.

Добавлена возможность по умолчанию указать <code>uuid</code> в качестве первичного ключа при генерации миграций базы данных.

Добавлены <code>ActiveRecord::Relation#left_joins</code> и <code>ActiveRecord::Relation#left_outer_joins</code>.

Добавлены колбэки <code>after_{create,update,delete}_commit</code>.

Версия API представлена в классах миграций, таким образом можно изменять значения по умолчанию без риска сломать существующие миграции, или принудить переписать их с помощью цикла устаревания.

<code>ApplicationRecord</code> - это новый суперкласс для всех моделей приложения, по аналогии с контроллерами приложения, являющимися подклассами <code>ApplicationController</code> вместо <code>ActionController::Base</code>. Это дает возможность приложениям иметь единое место для настройки специфичного для приложения поведения модели.

Добавлены методы ActiveRecord <code>#second_to_last</code> и <code>#third_to_last</code>.

Добавлена возможность аннотации объектов базы данных (таблиц, столбцов, индексов) комментариями, хранимыми в метаданных базы данных для PostgreSQL & MySQL.

Добавлена поддержка подготовленных выражений (prepared statements) для адаптера mysql2, для mysql2 0.4.4+. Раньше это поддерживалось только устаревшим адаптером mysql. Чтобы включить, установите <code>prepared_statements: true</code> в <code>config/database.yml</code>.

Добавлена возможность вызвать <code>ActionRecord::Relation#update</code> на реляционных объектах, который запустит валидации на колбэках на всех объектах в реляции.

Добавлена опция <code>:touch</code> в метод <code>save</code>, таким образом, записи могут быть сохранены без обновления временных меток.

Добавлена поддержка индексов по выражениям (expression indexes) и классов оператора (operator classes) для PostgreSQL.

Добавлена опция <code>:index_errors</code> для добавления индексов к ошибкам вложенных атрибутов.

## 7. Active Model
За подробностями обратитесь к <a href="https://github.com/rails/rails/blob/5-0-stable/activemodel/CHANGELOG.md" target="_blank">Changelog</a>.

### 7.1. Удалено

Удалены устаревшие <code>ActiveModel::Dirty#reset_#{attribute}</code> и <code>ActiveModel::Dirty#reset_changes</code>.

Удалена сериализация XML. Эта особенность была извлечена в гем <code>activemodel-serializers-xml</code>.

Удален модуль <code>ActionController::ModelNaming</code>.

### 7.2. Устарело

Устарел возврат <code>false</code> в качестве способа прервать цепочку колбэков <code>Active Model</code> и <code>ActiveModel::Validations</code>. Рекомендуемый способ <code>throw(:abort)</code>.

Устарели методы <code>ActiveModel::Errors#get</code>, <code>ActiveModel::Errors#set</code> и <code>ActiveModel::Errors#[]=</code>, имеющие противоречивое поведение.

Устарела опция <code>:tokenizer</code> для <code>validates_length_of</code> в пользу чистого Ruby.

Устарели <code>ActiveModel::Errors#add_on_empty</code> и <code>ActiveModel::Errors#add_on_blank</code> без замены.

### 7.3. Значимые изменения

Добавлен <code>ActiveModel::Errors#details</code> для определения, какие валидаторы провалились.

Извлечен <code>ActiveRecord::AttributeAssignment</code> в <code>ActiveModel::AttributeAssignment</code>, позволяя его использование в любом объекте в качестве включаемого модуля.

Добавлены <code>ActiveModel::Dirty#[attr_name]_previously_changed?</code> и <code>ActiveModel::Dirty#[attr_name]_previous_change</code> для улучшения доступа в записанные изменения после того, как модель была сохранена.

Валидация нескольких контекстов за раз в <code>valid?</code> и <code>invalid?</code>.

## 8. Active Job
За подробностями обратитесь к <a href="https://github.com/rails/rails/blob/5-0-stable/activejob/CHANGELOG.md" target="_blank">Changelog</a>.

### 8.1. Значимые изменения

<code>ActiveJob::Base.deserialize</code> делегируется в класс задачи. Это позволяет задачам присоединить произвольные метаданные при сериализации и прочитать их при выполнении.

Добавлена возможность настроить адаптер очереди для каждой задачи без взаимного влияния друг на друга.

Сгенерированная задача теперь по умолчанию наследуется от <code>app/jobs/application_job.rb</code>.

Позволяет <code>DelayedJob</code>, <code>Sidekiq</code>, <code>qu</code>, <code>que</code> и <code>queue_classic</code> возвращать <code>ActiveJob::Base</code> id задачи как <code>provider_job_id</code>.

Реализован простой процессор <code>AsyncJob</code> и связанный <code>AsyncAdapter</code>, который складывает задачи в пул тредов <code>concurrent-ruby</code>.

Изменен адаптер по умолчанию со встроенного на асинхронный. Это лучше по умолчанию, так как тогда тесты не будут ошибочно проходить, полагаясь на поведение, проходящее синхронно.

## 9. Active Support
За подробностями обратитесь к <a href="https://github.com/rails/rails/blob/5-0-stable/activesupport/CHANGELOG.md" target="_blank">Changelog</a>.

### 9.1. Удалено

Удален устаревший <code>ActiveSupport::JSON::Encoding::CircularReferenceError</code>.

Удалены устаревшие методы <code>ActiveSupport::JSON::Encoding.encode_big_decimal_as_string=</code> и <code>ActiveSupport::JSON::Encoding.encode_big_decimal_as_string</code>.

Удален устаревший <code>ActiveSupport::SafeBuffer#prepend</code>.

Удалены устаревшие методы из Kernel. <code>silence_stderr</code>, <code>silence_stream</code>, <code>capture</code> и <code>quietly</code>.

Удален устаревший файл <code>active_support/core_ext/big_decimal/yaml_conversions</code>.

Удалены устаревшие методы <code>ActiveSupport::Cache::Store.instrument</code> и <code>ActiveSupport::Cache::Store.instrument=</code>.

Удален устаревший <code>Class#superclass_delegating_accessor</code>. Вместо него используйте <code>Class#class_attribute</code>.

Удален устаревший <code>ThreadSafe::Cache</code>. Вместо него используйте <code>Concurrent::Map</code>.

Удален <code>Object#itself</code>, так как он реализован в Ruby 2.2.

### 9.2. Устарело

Устарел <code>MissingSourceFile</code> в пользу <code>LoadError</code>.

Устарел <code>alias_method_chain</code> в пользу <code>Module#prepend</code>, представленного в Ruby 2.0.

Устарел <code>ActiveSupport::Concurrency::Latch</code> в пользу <code>Concurrent::CountDownLatch</code> из concurrent-ruby.

Устарела опция <code>:prefix</code> для <code>number_to_human_size</code> без замены.

Устарел <code>Module#qualified_const_</code> в пользу встроенных методов <code>Module#const_</code>.

Устарела передача строки для определения колбэков.

Устарели <code>ActiveSupport::Cache::Store#namespaced_key</code>, <code>ActiveSupport::Cache::MemCachedStore#escape_key</code> и <code>ActiveSupport::Cache::FileStore#key_file_path</code>. Вместо них используйте <code>normalize_key</code>.

Устарел <code>ActiveSupport::Cache::LocaleCache#set_cache_value</code> в пользу <code>write_cache_value</code>.

Устарела передача аргументов в <code>assert_nothing_raised</code>.

Устарел <code>Module.local_constants</code> в пользу <code>Module.constants(false)</code>.

### 9.3. Значимые изменения

Добавлены методы <code>#verified</code> и <code>#valid_message?</code> в <code>ActiveSupport::MessageVerifier</code>.

Изменен способ, которым прерываются цепочки колбэков. Теперь предпочтительный метод прерывания цепочки колбэков – явный <code>throw(:abort)</code>.

Новая конфигурационная опция <code>config.active_support.halt_callback_chains_on_return_false</code> для определения, могут ли колбэки <code>ActiveRecord</code>, <code>ActiveModel</code> и <code>ActiveModel::Validations</code> быть прерваны, возвращая <code>false</code> в колбэке <code>'before'</code>.

Изменена сортировка тестов по умолчанию с <code>:sorted</code> на <code>:random</code>.

Добавлены методы <code>#on_weekend?</code>, <code>#on_weekday?</code>, <code>#next_weekday</code>, <code>#prev_weekday</code> в <code>Date</code>, <code>Time</code> и <code>DateTime</code>.

Добавлена опция <code>same_time</code> для <code>#next_week</code> и <code>#prev_week</code> в <code>Date</code>, <code>Time</code> и <code>DateTime</code>.

Добавлены аналоги <code>#prev_day</code> и <code>#next_day</code> для <code>#yesterday</code> и <code>#tomorrow</code> в <code>Date</code>, <code>Time</code> и <code>DateTime</code>.

Добавлен <code>SecureRandom.base58</code> для генерации случайных строк base58.

Добавлен <code>file_fixture</code> в <code>ActiveSupport::TestCase</code>. Он представляет простой механизм для доступа к файлам с примерами в ваших тестовых случаях.

Добавлен <code>#without</code> в <code>Enumerable</code> и <code>Array</code>, возвращающий копию <code>enumerable</code> без определенных элементов.

Добавлены <code>ActiveSupport::ArrayInquirer</code> и <code>Array#inquiry</code>.

Добавлен <code>ActiveSupport::TimeZone#strptime</code>, позволяющий парсить время, как будто из заданной временной зоны.

Добавлены предикатные методы <code>Integer#positive?</code> и <code>Integer#negative?</code> в духе <code>Integer#zero?</code>.

Добавлены восклицательные версии методов доступа в <code>ActiveSupport::OrderedOptions</code>, вызывающие <code>KeyError</code>, если значение <code>.blank?</code>.

Добавлен <code>Time.days_in_year</code>, возвращающий количество дней в заданном году, или в текущем году, если не указан аргумент.

Добавлен событийный мониторинг файлов для асинхронного обнаружения изменений в исходном коде приложения, маршрутах, локалях и так далее.

Добавлен набор методов <code>thread_m/cattr_accessor/reader/writer</code> для объявления переменных класса и модуля, существующих отдельно для каждого треда.

Добавлены методы <code>Array#second_to_last</code> и <code>Array#third_to_last</code>.

Добавлен метод <code>#on_weekday?</code> в <code>Date</code>, <code>Time</code> и <code>DateTime</code>.

Опубликованы API <code>ActiveSupport::Executor</code> и <code>ActiveSupport::Reloader</code>, чтобы позволить компонентам и библиотекам управлять и участвовать в выполнении кода приложения и процессе перезагрузки приложения.

Теперь <code>ActiveSupport::Duration</code> поддерживает форматирование и парсинг ISO8601.

В <code>TaggedLogging</code> добавлена возможность логгерам быть инициализированными несколько раз, и у них не будет общих тегов между собой.


