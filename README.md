# i18n-tasks [![Build Status](https://travis-ci.org/glebm/i18n-tasks.png?branch=master)](https://travis-ci.org/glebm/i18n-tasks) [![Coverage Status](https://coveralls.io/repos/glebm/i18n-tasks/badge.png?branch=master)](https://coveralls.io/r/glebm/i18n-tasks?branch=master) [![Code Climate](https://codeclimate.com/github/glebm/i18n-tasks.png)](https://codeclimate.com/github/glebm/i18n-tasks)


Tasks to manage translations in ruby applications using I18n.

![i18n-screenshot](https://raw.github.com/glebm/i18n-tasks/master/doc/img/i18n-tasks.gif "i18n-tasks output screenshot")

## Usage

Use `rake -T i18n` to get the list of tasks with descriptions. These are [the tasks](/lib/tasks/i18n-tasks.rake) available:

There are reports for missing and unused translations:

```bash
rake i18n:missing
rake i18n:unused
```

You can also export missing and unused data to XLSX though there is no importer yet:

```bash
rake i18n:spreadsheet_report
```

To remove unused translations run:

```bash
rake i18n:remove_unused # this will print the unused report and ask for confirmation before deleting keys
```

i18n-tasks can add missing keys to the locale data, and it can also fill untranslated values.

To add the keys that are not in the base locale but detected in the source do:

```bash
# add missing keys to the base locale data (I18n.default_locale)
# values set to key.humanize
rake i18n:add_missing
# or set all values to the task argument
rake i18n:add_missing[OhNoesMissing]
```

Prefill empty translations using Google Translate ([more below on the API key](#translation-config)).

```bash
rake i18n:fill:google_translate
# this task and the ones below can also accept specific locales:
rake i18n:fill:google_translate[es+de]
```

Prefill using values from the base locale - `I8n.default_locale`:
```bash
rake i18n:fill:base_value
```

i18n-tasks sorts the keys and writes them to their respective files:

```bash
# this happens automatically on any i18n:fill:* task
rake i18n:normalize 
```


`i18n:unused` will detect pattern translations and not report them, e.g.:

```ruby
t 'category.' + category.key # 'category.arts_and_crafts' considered used
t "category.#{category.key}" # also works
```

Relative keys (`t '.title'`) and plural keys (key.one/many/other/etc) are fully supported.

For more examples see [the tests](/spec/i18n_tasks_spec.rb).


## Installation

Simply add to Gemfile:

```ruby
gem 'i18n-tasks', '~> 0.2.10'
```

If not using Rails, require the tasks in Rakefile:

```ruby
# Rakefile
load 'tasks/i18n-tasks.rake'
```

## Configuration

Configuration is read from `config/i18n-tasks.yml` or `config/i18n-tasks.yml.erb`.

By default, `i18n-tasks` will work with `I18n.default_locale` and `I18n.available_locales`, but you can override this:

```yaml
# config/i18n-tasks.yml
base_locale: en
locales: [es, fr]
```

### Storage

```yaml
# i18n data storage
data:
  # The default YAML adapter supports reading from and writing to YAML files
  adapter: yaml
  # a list of file globs to read from per-locale
  read: 
    # this one is default:
    - 'config/locales/%{locale}.yml'
    # add this one to also read from namespaced files, e.g. simple_form.en.yml:
    - 'config/locales/*.%{locale}.yml'
  # a list of {key pattern => file} routes, matched top to bottom
  write:
    # this would save all devise keys in it's own file (per locale):
    - ['devise.*', 'config/locales/devise.%{locale}.yml']
    # this is the default catch-all:
    - 'config/locales/%{locale}.yml' # path is short for ['*', path]
```

Key matching syntax:

| syntax       | description                                               |
|:------------:|:----------------------------------------------------------|
|      `*`     | matches everything                                        |
|      `:`     | matches a single key                                      |
|   `{a, b.c}` | match any in set, can use `:` and `*`, match is captured  |

Example:

```yaml
data:
  write:
    # store sorcery and simple_form keys in the respective files:
    - ['{sorcery,simple_form}.*', 'config/locales/\\1.%{locale}.yml']
    # write every namespace to its own file:
    - ['{:}.*', 'config/locales/\1.%{locale}.yml']
```

### Usage search

```yaml
# i18n usage search in source
search:
  # search these directories (relative to your Rails.root directory, default: 'app/')
  paths:
    - 'app/'
    - 'vendor/'
  # include only files matching this glob pattern (default: blank = include all files)
  include:
    - '*.rb'
    - '*.html.*'
    - '*.text.*'
  # explicitly exclude files (default: blank = exclude no files)
  exclude:
    - '*.js'
  # you can override the default key regex pattern:
  pattern: "\\bt[( ]\\s*(:?\".+?\"|:?'.+?'|:\\w+)"
```

To configure paths for relative key resolution:

```yaml
# directories containing relative keys
relative_roots:
  # default:
  - app/views
  # add a custom one:
  - app/views-mobile
```


### Fine-tuning

Tasks may incorrectly report framework i18n keys as missing, also some patterns may not be detected.
When all else fails, use the options below.

```yaml
# do not report these keys as unused
ignore_unused:
  - category.*.db_name

# do not report these keys as missing (both on blank value and no key)
ignore_missing:
  - devise.errors.unauthorized # ignore this key
  - pagination.views.*         # ignore the whole pattern
  # E.g to ignore all Rails number / currency keys:
  - 'number.{format, percentage.format, precision.format, human.format, currency.format}.{strip_insignificant_zeros,significant,delimiter}'
  - 'time.{pm,am}'

# do not report these keys when they have the same value as the base locale version
ignore_eq_base:
  all:
    - common.ok
  es,fr:
    - common.brand

# do not report these keys ever
ignore:
  - kaminari.*
```

<a name="translation-config"></a>
### Google Translate

`rake i18n:fill:google_translate` requires a Google Translate API key, get it at [Google API Console](https://code.google.com/apis/console).
Put the key in `GOOGLE_TRANSLATE_API_KEY` environment variable or in the config file.

```yaml
translation:
  api_key: <Google Translate API key>
```

## RSpec integration

You might want to test for missing and unused translations as part of your test suite.
This is how you can do it with rspec:

```ruby
# spec_helper.rb
require 'i18n/tasks'
require 'i18n/tasks/base_task'

# spec/locales_spec.rb
require 'spec_helper'
describe 'translations'  do
  let(:i18n) { I18n::Tasks::BaseTask.new }

  it 'are all used' do
    i18n.unused_keys.should have(0).keys
  end

  it 'are all present' do
    i18n.untranslated_keys.should have(0).keys
  end
end
```

## HTML report

While i18n-tasks does not provide an HTML version of the report, it's easy to roll your own, see [the example](https://gist.github.com/glebm/6887030).

---

This was originally developed for [Zuigo](http://zuigo.com/), a platform to organize and discover events.

[MIT license](/LICENSE.txt)


[![Bitdeli Badge](https://d2weczhvl823v0.cloudfront.net/glebm/i18n-tasks/trend.png)](https://bitdeli.com/free "Bitdeli Badge")

