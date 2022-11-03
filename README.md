# fastfile-ios

- [Lane List](#lane-list)
- [Version Midset](#version-midset)
- [Installation](#installation)
- [Release Guide](#release-guide)
- [Develop Guide](#develop-guide)

## Lane List
### Basic
- build：給 CI 跑的，驗證專案在 CI Server 上也跑得起來。
- unit_test：跑 unit test。
	- without_build (true/false)

### Version Number
- bump_build_number：build number 跳一號，會去檢查 TestFlight 跟 AppStore 上面的 build number，避免比上面的值還低。
	- number：指定數字。
- bump_version：更新版號，可以決定要跳的是 major、minor 還是 patch。
	- version：指定版本。

### Archive
- alpha
	- distribute_method: TestFlight（預設）、Firebase
- beta
- alpha_beta
- alpha_beta_release
- release
- daily_archive：給 CI 跑排程用的，就是跑 `alpha`，差別在於會先檢查跟上一版比有沒有新的功能或修正，用 git message 檢查。

裡面都會包含以下功能：
1. 跳一版 build number
2. Archive
3. 上傳 dsym
3. 讀取 change log
4. 上傳 .ida
5. 更新 change log 壓時間

### Library
#### install 
- install_library：安裝 Cocoapods、Carthage 相關套件。
- carthage_install：因為 Carthage 的指令比較複雜，所以包成 lane 方便日常使用。
- carthage_update：因為 Carthage 的指令比較複雜，所以包成 lane 方便日常使用。
	- library：library 的名字。

#### update
- update_library：更新所有相關套件，包含：Bundler、CocoaPods、Carthage 的。設計給 CI 定期跑的，設計理念是避免太久更新版本，一次錯一大堆，還不如定期更新套件，build 不起來就知道有新版版相容的問題了，就看開發者想不想要這樣使用。
- fastlane_update：獨立只更新 fastlane，因為 fastlane 更新相對來說是比較沒有問題。

### iTunes Connect
- upload_ipa：上傳 .ipa。從 archive 裡面開放出來，因為有時是用 GUI 打包，可能是指令打包出問題，這時我們已經有 .ipa 了，所以只要上傳就可以了。
	- distribute_method: TestFlight（預設）、Firebase
- screenshots：產生上架要用的螢幕截圖並上傳。
	- devices(array)：指定要哪些裝置的，非必要已經有預設值了。
- update_snapshot：單純上傳螢幕截圖。

#### 為了不想在網頁上開 iTunesConnect，因為登入、頁面切換都很花時間
- create_new_app：開一個全新的 App。
- update_meta_data：只更新送審資料。
- submit_for_review：送審成功的話發送 Slack。

### Others
- add_device：增加新的測試裝置，並且更新 provision profile。
- generate_badge_icon：在 icon 上加上 badge。
- readme_auto_update：用來 commit fastlane readme 的。如果 fastfile 有改，fastlane 的 readme 也會更新。
- send_notification：發出 slack 通知。從原本的 fastfile 開放出來，如果有原本設計以外的需求就可以用。
	- title
	- message：沒有一定要設定，如果跟 title 都有設定，title 就會自動加粗體。
	- inform_level：developer、product_manager。
	- success：true、false。用來表示在 CI 跑的事情成功還是失敗，成功顯示綠色，失敗顯示紅色。

## Version Midset

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


## Release Guide

This Fastfile use `fastlane-plugin-changelog` to read and write changelog. First you need to create a `CHANGELOG.md` at project root folder.

### Archive for alpha/beta version

add a new block in change log

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

### 特殊套件解釋

* tty-prompt：提供簡潔的 API 來讀取使用的輸入的值。
* dotenv：用 .env 的方式讀取寫入 ENV。
* envied：用來安全的讀取布林值，'0'/'1'、'f'/'t'、'false'/'true'、'off'/'on'、'no'/'yes' 都支援。
* fastlane-plugin-versioning：更新版號的 plugin，原生的 action 會有問題。
