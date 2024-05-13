# Custom Rules

A custom rule is any rule that is not maintained by Elastic under `rules/` or `rules_building_block`. These docs are intended
to show how to manage custom rules using this repository.

For more detailed breakdown and explanation of employing a detections-as-code approach, refer to the
[dac-reference](https://dac-reference.readthedocs.io/en/latest/index.html).


## Defining a custom rule config and directory structure

The simplest way to maintain custom rules alongside the existing prebuilt rules in the repo, is to decouple where the rules
are stored to minimize VCS conflicts and overlap. This is accomplished by defining a custom rules directory using a config file.

### Understanding the structure

```
custom-rules
├── _config.yaml
└── rules
    ├── example_rule_1.toml
    ├── example_rule_2.toml
└── etc
    ├── deprecated_rules.json
    ├── packages.yml
    ├── stack-schema-map.yaml
    ├── test-config.yml
    └── version.lock.json
└── actions
    ├── action_1.toml
    ├── action_2.toml
└── exceptions
    ├── exception_1.toml
    └──  exception_2.toml
```

This structure represents a portable set of custom rules. This is just an example, and the exact locations of the files
should be defined in the `_config.yaml` file. Refer to the details in the default
[_config.yaml](../detection_rules/etc/_config.yaml) for more information.

* deprecated_rules.json - tracks all deprecated rules (optional)
* packages.yml - information for building packages (mostly optional, but the current version is required)
* stack-schema-map.yaml - a mapping of schemas for query validation
* test-config.yml - a config file for testing (optional)
* version.lock.json - this tracks versioning for rules (optional depending on versioning strategy)

To initialize a custom rule directory, run `python -m detection_rules custom-rules setup-config <directory>`

### Defining a config

```yaml
rule_dirs:
  - rules
  - rules_building_block
files:
  deprecated_rules: deprecated_rules.json
  packages: packages.yml
  stack_schema_map: stack-schema-map.yaml
  version_lock: version.lock.json
directories:
  actions_dir: actions
  exceptions_dir: exceptions
```

* Note: the paths in this file are relative to the custom rules directory (CUSTOM_RULES_DIR/)
* Note: Refer to each original source file for purpose and proper formatting

When using the repo, set the environment variable `CUSTOM_RULES_DIR=<directory-with-_config.yaml>`


### Defining a testing config

```yaml
testing:
  config: etc/example_test_config.yaml
```

This points to the testing config file (see example under detection_rules/etc/example_test_config.yaml) and can either
be set in `_config.yaml` or as the environment variable `DETECTION_RULES_TEST_CONFIG`, with precedence going to the
environment variable if both are set. Having both these options allows for configuring testing on prebuilt Elastic rules
without specifying a rules _config.yaml.


* Note: If set in this file, the path should be relative to the location of this config. If passed as an environment variable, it should be the full path


### How the config is used and it's designed portability

This repo is designed to operate on certain expectations of structure and config files. By defining the code below, it allows
the design to become portable and based on defined information, rather than the static excpectiations.

```python
RULES_CONFIG = parse_rules_config()

# which then makes the following attribute available for use

@dataclass
class RulesConfig:
    """Detection rules config file."""
    deprecated_rules_file: Path
    deprecated_rules: Dict[str, dict]
    packages_file: Path
    packages: Dict[str, dict]
    stack_schema_map_file: Path
    stack_schema_map: Dict[str, dict]
    version_lock_file: Path
    version_lock: Dict[str, dict]
    test_config: TestConfig

    action_dir: Optional[Path] = None
    exception_dir: Optional[Path] = None

# using the stack_schema_map
RULES_CONFIG.stack_schema_map
```


### Custom actions and exceptions lists

To convert these to TOML, you can do the following:

1. export the ndjson from Kibana into a `dict` or load from kibana

```python
from detection_rules.action import Action, ActionMeta, TOMLActionContents, TOMLAction

action = Action.from_dict(action_dict)
meta = ActionMeta(...)
action_contents = TOMLActionContents(action=[action], meta=meta)
toml_action = TOMLAction(path=Path, contents=action_contents)
```

Mimick a similar approach for exception lists. Both can then be managed with the `GenericLoader`

```python
from detection_rules.generic_loader import GenericLoader

loader = GenericLoader()
loader.load_directory(...)
```