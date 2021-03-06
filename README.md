# amqcfg - AMQ configurator

This tool can generate_via_tuning_files a set of configuration files mainly needed for
AMQ Broker, but it is not limited to only generating files for one product.

It has a user facing Command Line Tool for quick and easy command line usage.
Furthermore, it is possible to use its API in your python code.

## Getting started

* python 3.5+ or python2.7
* current requirements from setup.py (runtime requirements only)
* python virtualenv recommended (install via system package manager
or `pip install --user virtualenv`)

for contributors:
* requirements from requirements.txt (there are Dev and QA requirements as well)

### From git

```bash
git clone git@bitbucket.org:msgqe/amqcfg.git
python -m virtualenv -p python3 venv3
source venv3/bin/activate
./setup.py install
amqcfg --help
```

### From PiPy

```bash
python -m virtualenv -p python3 venv3
source venv3/bin/activate
pip install amqcfg
amqcfg --help
```

## User (CLI) guide

```bash
amqcfg --help

amqcfg --list-profiles
amqcfg --list-templates

# perform a generation of a default profile
amqcfg --profile artemis/2.5.0/default.yaml
# also save result to [OUTDIR] directory
amqcfg --profile [PROFILE] --output [OUTDIR]
```

## Customization

Quickest way to customize data is to use hot-variables, basically variables
that the profile itself provides for tuning. Next step is to write (modify) custom
profile with completely custom values.
If that does not satisfy your needs, then a custom template might be required.

### Profile tuning

Simply export tuning values from profile you want to tune and change those you
need to change. Then supply the custom tuning file(s) when generating the profile.

```bash
amqcfg --profile [PROFILE] --export-tuning my_values.yaml
vim my_values.yaml
amqcfg --profile [PROFILE] --tune my_values.yaml

# multiple tuning files can be overlaid
# they are updated in sequence, only values present are overwritten
amqcfg --profile [PROFILE] --tune my_values.yaml --tune machine_specific.yaml \
       --tune logging_debug.yaml --output [OUTDIR]
```

## Custom profiles

Write your own, or simply export an existing profile and modify that.

You can export dynamic version with includes of some modules, that would still
 work. Either you can use imports from package, or your own local files.

Or you can export completely rendered profile file without any includes or
variables and modify that as you like.


```bash
# export profile with dynamic includes
amqcfg --profile [PROFILE] --new-profile my_new_profile.yaml
# export completely generated profile
amqcfg --profile [PROFILE] --new-profile-static my_new_profile.yaml
vim my_new_profile.yaml
amqcfg --profile my_new_profile.yaml
```

## Custom templates

The last resort is to export a template and modify that. But remember a template,
or more correctly a template set is a directory containing a set of main
templates that subsequently generate_via_tuning_files a new file.

Of course feel free to write your own templates. Especially when you need to
generate_via_tuning_files files for something that is not packaged.

Just remember for a template set to be identified the directory must contain
a file named '_template' and then main templates ending with '.jinja2'.

```bash
amqcfg --template [TEMPLATE] --new-template my_new_template
vim my_new_template/[MAIN_TEMPLATES].jinja2
amqcfg --template my_new_template --profile [PROFILE]

```

## API guide

Direct use of API is to use `generate()` nearly the same as the CLI.
With option to use tuning values directly.

Tuning data will be overlaid in order of appereance, using python
dict.update(), so values that will appear later will overwrite previous
values. We recommend that tuning values are always flat, because update
is not recursive. The same applies for data from tuning files as well
as the directly provided data.

Data application order:

- profile defaults
- data from tuning files (in order of appearance) `tuning_files_list`
- data provided directly (in order of appearance) `tuning_data_list`


```python
import amqcfg

# generating only broker.xml config using default values from profile,
# no tuning, writing output to a target path
amqcfg.generate(
    profile='artemis/2.5.0/default.yaml',
    output_filter=['broker.xml'],
    output_path='/opt/artemis-2.5.0-i0/etc/',
)

# using both files and direct values, and writing generated configs to
# a target directory
amqcfg.generate(
    profile='artemis/2.5.0/default.yaml',
    tuning_files_list=[
        'my_values.yaml',
        'machine_specific.yaml',
        'logging_debug.yaml'
    ],
    tuning_data_list=[
        {'name': 'custom name', 'config': 'option_a'},
        {'address': '10.0.0.1'},
        {'LOG_LEVEL': 'debug'},
    ],
    output_path='/opt/artemis-2.5.0-i0/etc/',
)

# just get generated data for further processing, using just tuning files
data = amqcfg.generate(
    profile='artemis/2.5.0/default.yaml',
    tuning_files_list=[
        'my_values.yaml',
        'machine_specific.yaml',
        'logging_debug.yaml'
    ],
)
print(data['broker.xml'])
```

## Batch configurations

In case you have multiple services to configure in your environment,
that you probably will have at some point, there is a tool for that
as well. The tool is called amqcfg-batch. It has only yaml input and
it uses amqcfg to generate configurations as you are already used to.

Input yaml file defines all services you need to generate, what
profiles to use, and what tuning to provide to `amqcfg`.
It allows you to configure defaults and common for services.

### Batch profile file

As said it is YAML. It has two special sections: `_default` and `_common`.
As the name suggests, `_default` values are used when values are not
defined per specific section. Where `_common` is added to the values
of all sections. The important thing here is that `_default` has lower
priority than `_common` and that has lower priority than specific section
values.

Every section has 4 values: `profile`, `template`, `tuning_files`,
 and `tuning`. As the name suggests, `profile` defines what generation profile
 to select, and it directly correlates with `amqcfg`'s `--profile`.
 `template` defines what generation template to use
 (overrides one in the profile if defined), and it directly correlates with
 `--template` from `amqcfg`. `tuning_files` option is a list of tuning
 files to use, when combining defaults, commons, and specific values,
 tuning_files list is concatenated. Finally `tuning` is a map of
 specific tuning values, correlates with `--opt` of `amqcfg`. When combining
 defaults, commons, and specifics, it will be updated over using python
 dict.update() and it will work only on first level, so it is recommended
 to use flat values for tuning only.

Example:
```yaml

_default:
    profile: artemis/2.5.0/default.yaml
    tuning_files:
      - defaults/broker_default.yaml

_common:
    tuning_files:
      - common/security.yaml
      - common/logging.yaml
    tuning_values:
      LOG_LEVEL_ALL: INFO

brokerA/opt/artemis/etc:
    pass: true

brokerB/opt/artemis/etc:
    profile: artemis/2.5.0/AIOBasic.yaml
    tuning_files:
      - brokerB/queues.yaml

---

_default:
    profile: amq_broker/7.2.0/default.yaml
    tuning_files:
      - defaults/amq_broker_default.yaml

brokerC/opt/amq/etc:
    tuning:
      LOG_LEVEL_ALL: DEBUG

```

As you can see, `amqcfg-batch` supports multiple sections, in single
batch profile file, that allows you to generate multiple groups using
separated `_default` and `_common` sections for that.

### executing batch

When you have defined all tuning files you need, and in the root of this
batch configuration you have your batch profile file, you can now simply
run `amqcfg-batch`:

```bash

amqcfg-batch --input [batch_profile_file] --output [output_path]
```

You can use multiple input files and all of those will be generated
consecutively. In the output path, new subdirectories will be created
for every item you configure (every section), section key will be used
for that subdirectory. If the section name resembles a path, whole
path will be created. For example for `brokerA/opt/artemis/etc`
the configuration will be generated into
`[output_path]/brokerA/opt/artemis/etc/`.

## Documentation

this readme and docstrings, for now. Sorry about that.

I would like to have [readthedocs.org](http://readthedocs.org) documentation.

## Contributing

If you find a bug or room for improvement, submit either a ticket or PR.

## Contributors

_Alphabetically ordered_

* Michal Tóth <mtoth@redhat.com>
* Otavio Piske <opiske@redhat.com>
* Sean Davey <sdavey@redhat.com>
* Zdenek Kraus <zkraus@redhat.com> (maintainer)

## License

Copyright 2018 Red Hat Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

## Acknowledgments

* [jinja2](http://jinja.pocoo.org/docs/2.10/) -- awesome templating engine
* [yaml](http://yaml.org/) -- very convenient user readable format
* [learn_yaml](https://learnxinyminutes.com/docs/yaml/) -- great YAML cheat sheet
* [pyyaml](https://github.com/yaml/pyyaml) -- python YAML parser
* [jq](https://stedolan.github.io/jq/) -- great tool for working with structured data (JSON)
* [yq](https://yq.readthedocs.io/en/latest/) -- YAML variant of jq
