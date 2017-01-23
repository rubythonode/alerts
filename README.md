# alerts example #
This is an example alerts repository for interferon which adds a few alerts and groups as well as adding a couple of custom host sources.

## Table of Contents

* [Customizations](#customizations)
  * [config.yml](#configyml)
  * [alerts](#alerts)
  * [destinations](#destinations)
  * [groups](#groups)
  * [group_sources](#group_sources)
  * [host_sources](#host_sources)
* [Tests](#tests)
* [Deploying](#deploying)

## Customizations
There are a few customization points for interferon that can be implemented in the alerts repository without having to commit changes to interferon itself. With the exception of `config.yml` the other entries in this list are all directories.

### config.yml
The `config.yml` file contains settings for interferon including which components to enable. It has the following configurable settings that can be set as YAML mapping keys:

* **`verbose_logging`**: boolean value `true` or `false` which sets the logging verbosity for interferon.
* **`alerts_repo_path`**: string value of the directory to look for the `alerts` directory and any additional directories for `destinations`, `group_sources`, `host_sources` modules.
* **`destinations`** / **`group_sources`** / **`host_sources`**: list of maps containing the following keys.
  * **`type`**: string value of the module to load for the given type (e.g. "filesystem" will look for `filesystem.rb` in the directory named as the current parent key in the `alerts_repo_path`).
  * **`enabled`**: boolean value `true` or `false` whether or not to use a particular module.
  * **`options`**: map of additional options to pass to the module.

### alerts
You can look at the example files in `alerts/` to get a taste of the syntax. You will need the following fields in your alerts file:

* **`name`**: this will be the primary key of your alert, and will show up in emails and in the UI. Only one alert of a given name will be created, so if your alert name is not using some part of the `@hostinfo` object to make it unique, you will end up with a single alert.
* **`message`**: the body of the alert that will be set to the destinations (currently only Datadog).
* **`applies`**: determines which hosts the alert applies to; putting `true` will end up creating as many alerts as the number of distinct `name`s your alert expands into. Interferon will iterate through all the hosts generated by `host_sources` and test them agains the expression here.
* **`notify.people`**: an array of the people who will be receiving the alert.
* **`notify.groups`**: an array of groups who will be receiving the alert. The group mapping will be based on the output of `group_sources`. The `filesystem` group_source module will read the groups in the `groups` directory.
* **`metric`**: The Datadog metric alert query.
* **`silenced`**: The Datadog silenced setting.
* **`timeout`**: The Datadog timeout setting in seconds (gets converted to `timeout_h`).

### destinations
This directory contains additional modules for `destinations`.

### groups
This is the default location where the the built-in `filesystem` group_source reads it's YAML files.  If you don't want to store groups as YAML files or have your own group_source, you can remove this directory and remove the `filesystem` group_source from `config.yml`.

Otherwise, you may configure groups with YAML files containing the following keys:
* **`name`**: Name of the group (used in alerts).
* **`people`**: List of people in the group.
* **`alias_for`**: Name of another group that this group is an alias for. Useful for transistion periods when renaming groups.

To post to a slack channel, make sure that the slack integration is configured to include that channel: https://app.datadoghq.com/account/settings#integrations/slack

### group_sources
This directory contains additional modules for `group_sources`. Each group_source should be a class which implements the following methods.
* **`intialize(options)`**: Called for initializing the group_source and reading the options passed in from `config.yml`.
* **`list_groups`**: Called to get the group mapping generated by this module. Should return a hash contain group name to people.

### host_sources
This directory contains additional modules for `host_sources`.

We've implemented a few AWS-related host sources using [billow](https://github.com/airbnb/billow).
Using Billow instead of hitting the AWS API directly can help avoid hitting rate limits if deployments are frequent.

You can implement your own host_sources using your own APIs to expose the hosts available in your infrastructure.
Each host_source should be a class which implements the following methods:
* **`intialize(options)`**: Called for initializing the group_source and reading the options passed in from `config.yml`.
* **`list_hosts`**: Called to get the host list generated by this module. Should return a list of hashes that can be referenced using `@hostinfo` in alerts.

### Tests
Tests can be run via run via `rake spec` (to test additional code/modules) and `rake test` (to test the alert files). There is a datadog syntax checker built into `rake test` that will check for valid Datadog syntax.

There are additional tests in `script/pre-commit` and `test/check_syntax.sh` that can be run to check the syntax of other files.

### Deploying
When committing to the alerts repository, you should ensure that `rake test` is run to check the syntax of the datadog metrics.

To build the alerts repository run:
```
bundle install
```

Once built and deployed, interferon can be run via the following commands assuming the `config.yml` file is placed in the current working directory.

For dry-run:
```
bundle exec interferon --dry-run -c ./config.yml
```

For real run:
```
bundle exec interferon -c ./config.yml
```

#### Best practices
Although you can run interferon from your local workstation, we do not recommend this except for dry-run.
Since a single interferon instance expects to be managing all interferon created alerts, running multiple instances can cause a lot of churn if the alerts or any of the sources are not completely in sync.
You should use a build system which produces artifacts of this repository then have a deployment system set up the latest artifact and invoke `interferon`.
We also recommend running `interferon` periodically (i.e. using cron) on the deployed host to keep the alerts in sync with infrastructure changes and to clean up manual changes made in the UI.