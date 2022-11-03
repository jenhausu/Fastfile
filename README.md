# fastfile-ios

- [Installation](Installation)
- [Version](version)
- [Release Guide](release-guide)
- [Develop Guide](develop-guide)

## Installation

1. Add these line at the first of your `Fastfile`.

```
import_from_git(url: 'git@github.com:OsenseTech/fastfile-ios.git', path: 'Fastfile')
```

2. add these gems in your `Gemfile`, and then run `bundle install`

```
gem "fastlane"
gem "tty-prompt"
gem "envied"
gem "dotenv"
gem "cocoapods"
```

3. add these gems in your `Pluginfile`, and then run `bundle exec fastlane install_plugins`

```
gem 'fastlane-plugin-changelog'
gem 'fastlane-plugin-badge'
gem 'fastlane-plugin-versioning'
```

4. use it.

```
bundle exec fastlane [lane name]
```
Just use whatever lane you want. There is a `setEnvIfNeeded` method that will guide you on what ENV is needed.



## Version

The versions in this fastfile follow these mindset below.

|                 | dev                           | alpha                           | beta                                   | release                       |
|-----------------|-------------------------------|---------------------------------|----------------------------------------|-------------------------------|
| Purpose         | develop                       | internal test                   | external test / test production Server | AppStore                      |
| User            | developer                     | QA                              | people out side company                | end user                      |
| Feature         | not developed feature         | new feature                     | feature ready to release               | X                             |
| Stability       | very unstable                 | may have some bug               | all known bugs were fixed              | X                             |
| backend         | Test Server                   | Test Server                     | Production Server                      | Production Server             |
| Mixpanel        | Test                          | Test                            | Test                                   | Production                    |
| Scheme Name     | not specific                  | Alpha                           | Beta                                   | Release                       |
| bundle id       | com.[company_id].[app_id].dev | com.[company_id].[app_id].alpha | com.[company_id].[app_id].beta         | com.[company_id].[app_id].dev |
| Archive Timing  | X                             | Daily                           | Weakly/when needed                     | for review                    |
| App icon        | dev                           | alpha                           | beta                                   | X                             |
| Deploy Platform | Test Device                   | TestFlight / Adhoc              | TestFlight                             | AppStore                      |


## Release Guide

This Fastfile use `fastlane-plugin-changelog` to read and write changelog. First you need to create a `CHANGELOG.md` at project root folder.

### Archive for alpha/beta version

1. add a new block in change log

```
### [New Build]
```

### Archive for release version

add a new block in change log

```
### [New Release]
```


## Develop Guide


1. git clone these repo at the same level of your project.
2. Change import action to these, then fastfile will improt from local.
 
```
import "../../fastfile-ios/Fastfile"
```
