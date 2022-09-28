require 'envied'

default_platform(:ios)

ENV["FASTLANE_DONT_STORE_PASSWORD"] = "1"
ENV["NOT_WORK_ON_CI"] = "false"
ENV["USE_AUTO_SIGN"] = "false"

before_all do |lane, options|
    if options[:fake_lane] != nil
        ENV["FASTLANE_LANE_NAME"] = options[:fake_lane]
    end
    setup_ci

    begin
        setupEnv("APP_STORE_CONNECT_API_ISSUER_ID", "app store connet api issuer id", false)
        setupEnv("APP_STORE_CONNECT_API_KEY_ID", "app store connect api key id", false)
        setupEnv("APP_STORE_CONNECT_API_P8_FILEPATH", "app store connect api p8 filepath(./fastlane/AuthKey_XX2294X7XX.p8)", false)

        ensure_env_vars(
            env_vars: ['APP_STORE_CONNECT_API_KEY_ID', 'APP_STORE_CONNECT_API_ISSUER_ID', 'APP_STORE_CONNECT_API_P8_FILEPATH']
        )

        app_store_connect_api_key(
            key_id: ENV["APP_STORE_CONNECT_API_KEY_ID"],
            issuer_id: ENV["APP_STORE_CONNECT_API_ISSUER_ID"],
            key_filepath: ENV["APP_STORE_CONNECT_API_P8_FILEPATH"]
        )
    rescue
        puts 'use normal login'
    end

    begin
        # better way to access bool environment variable
        ENVied.require
    rescue
        sh("touch Envfile")
    end

    unless is_ci
        Dotenv.load('.env.local')
    end
end

def setupEnv(key, description, required = true)
    if ENV[key] == nil
        fastlane_require "tty-prompt"
        prompt = TTY::Prompt.new

        value = prompt.ask("#{description}:", required: required)
        if value != nil
            ENV[key] = value
            sh("echo '#{key}=#{value}' >> .env")
        end
    end
end

desc "Create a new app on iTunesconnect."
lane :create_new_app do
    fastlane_require "tty-prompt"
    prompt = TTY::Prompt.new
    setupEnv("BUNDLE_ID", "bundle id")
    setupEnv("PROJECT_NAME", "project name")

    app_identifier = ENV["BUNDLE_ID"]
    if ENV["BUNDLE_ID"] == nil
        app_identifier = prompt.ask("App Identifier: ", required: true)
    end

    app_name = ENV["PROJECT_NAME"]
    if ENV["PROJECT_NAME"] == nil
        app_name = prompt.ask("App Name", required: true)
    end

    produce(
        app_identifier: app_identifier,
        app_name: app_name
    )
end

desc "Build the project."
lane :build do
    if ENVied.NOT_WORK_ON_CI
        next
    end
    setupEnv("SCHEME_DEV", "development scheme")
    
    match(readonly: true) unless ENVied.USE_AUTO_SIGN
    install_library
    gym(
        scheme: ENV["SCHEME_DEV"],
        export_method: "app-store",
        cloned_source_packages_path: "Packages",
        silent: true
    )
    slack_message("Build Successfully", "developer", true)
end

desc "Run unit test."
lane :unit_test do |options|
    setupEnv("SCHEME_TEST", "test scheme")

    install_library
    scan(
        scheme: ENV["SCHEME_TEST"],
        device: "iPhone 13",
        test_without_building: options[:without_build],
        cloned_source_packages_path: "Packages"
    )
    slack_message("Unit Test Passed", "developer", true)
end

# Bump Version

desc "Bump build number."
lane :bump_build_number do |options|
    setupEnv("TARGET_NAME", "target name")
    setupEnv("BUNDLE_ID", "bundle id")
    setupEnv("BUNDLE_ID_ALPHA", "alpha bundle id", false)
    setupEnv("PROJECT_NAME", "project name")

    build_number = get_build_number.to_i

    if options[:number]
        build_number = options[:number]
    else
        # check testflight latest build number
        version = get_version_number(target: ENV["TARGET_NAME"])
        app_identifier_alpha = ENV["BUNDLE_ID_ALPHA"]
        if app_identifier_alpha == nil
            app_identifier_alpha = "#{ENV["BUNDLE_ID"]}.alpha"
        end

        begin
            testflight_build_number = latest_testflight_build_number(
                app_identifier: app_identifier_alpha,
                version: version,
                initial_build_number: 0
            )
            if testflight_build_number > build_number then
                build_number = testflight_build_number.to_i
            end
        rescue
            puts('not testflight')
        end

        begin
            # check appstore latest build number
            appstore_build_number = app_store_build_number(
                live: false,
                version: version,
                app_identifier: ENV["BUNDLE_ID"],
                initial_build_number: 0
            )
            if appstore_build_number > build_number then
                build_number = appstore_build_number
            end
        rescue
            puts('not appstore app')
        end
        build_number = build_number + 1
    end

    increment_build_number({
        build_number: build_number
    })
    
    if is_ci then
        sh(command: "git config --global user.name #{ENV["CI_NAME"]}")
    end
    commit_version_bump(
        message: "version[build]: #{get_build_number}",
        xcodeproj: "./#{ENV["PROJECT_NAME"]}.xcodeproj"
    )

    git_push
end

desc "Bump version with interactive command line mode."
lane :bump_version do |options|
    setupEnv("TARGET_NAME", "target name")
    setupEnv("PROJECT_NAME", "project name")

    if options[:version]
        increment_version_number(
            version_number: options[:version]
        )
        # in order to change market version
        increment_version_number_in_xcodeproj(
            version_number: options[:version]
        )
        increment_build_number(
            build_number: "0"
        )
        commit_version_bump(
            message: "version: #{(get_version_number(target: ENV["TARGET_NAME"]))}",
            xcodeproj: "./#{ENV["PROJECT_NAME"]}.xcodeproj"
        )
    else
        fastlane_require "tty-prompt"
        prompt = TTY::Prompt.new
        type = prompt.select("Which type of version to bump?", cycle: true) do |menu|
              menu.choice 'major'
              menu.choice 'minor'
              menu.choice 'patch'
        end

        increment_version_number(
            bump_type: type
        )
        # in order to change market version
        increment_version_number_in_xcodeproj(
            bump_type: type,
            target: ENV["TARGET_NAME"]
        )
        increment_build_number(
            build_number: "0"
        )
        commit_version_bump(
            message: "version[#{type}]: #{(get_version_number(target: ENV["TARGET_NAME"]))}",
            xcodeproj: "./#{ENV["PROJECT_NAME"]}.xcodeproj"
        )
    end

    if options[:tag]
        add_git_tag(
            tag: "#{get_version_number(target: ENV["TARGET_NAME"])}"
        )
    end
end

# Archive

lane :alpha do
    if have_new_commit
        bump_build_number
    end
    archive("Alpha")
    upload_api
    changelog = get_changelog
    changelog_update

    message = "✈️ Successfully deliver a new alpha version to TestFlight! (ﾉ>ω<)ﾉ ✈️"
    if ENV["ALPHA_RELEASE_MESSAGE"]
      message = ENV["ALPHA_RELEASE_MESSAGE"]
    end

    if changelog != ""
        pretext = "此版更新內容：\n#{changelog}"
    end

    slack_message(message, pretext, "product_manager", true)
end

lane :adhoc do
    if have_new_commit
        bump_build_number
    end
    archive("Alpha", "ad-hoc")
    upload_api(distribute_method: "Firebase")
    changelog = get_changelog
    changelog_update

    message = "Successfully deliver a new version to Firebase! (ﾉ>ω<)ﾉ"
    if ENV["ALPHA_RELEASE_MESSAGE"]
      message = ENV["ALPHA_RELEASE_MESSAGE"]
    end

    if changelog != ""
        pretext = "此版更新內容：\n#{changelog}"
    end

    slack_message(message, pretext, "product_manager", true)
end

lane :beta do
    if have_new_commit
        bump_build_number
    end
    archive("Beta")
    upload_api
    changelog = get_changelog
    if changelog != ""
        pretext = "此版更新內容：\n#{changelog}"
    end
    changelog_update
    slack_message("TestFlight 上有新的 beta 版本", pretext, "product_manager", true)
end

lane :alpha_beta do 
    if have_new_commit
        bump_build_number
    end
    archive("Alpha")
    upload_api
    archive("Beta")
    upload_api
    changelog = get_changelog
    if changelog != ""
        pretext = "此版更新內容：\n#{changelog}"
    end
    changelog_update
    slack_message("TestFlight 上有新的 Alpha, Beta 版本", pretext, "product_manager", true)
end

lane :alpha_beta_release do 
    if have_new_commit
        bump_build_number
    end
    archive("Alpha")
    upload_api
    archive("Beta")
    upload_api
    archive("Release")
    upload_api
    changelog = get_changelog
    if changelog != ""
        pretext = "此版更新內容：\n#{changelog}"
    end
    changelog_update
    slack_message("TestFlight 上有新的 Alpha, Beta, Release 版本", pretext, "product_manager", true)
end

lane :release do
    setupEnv("TARGET_NAME", "target name")

    if have_new_commit
        bump_build_number
    end
    archive("Release")
    upload_api
    changelog = get_changelog
    if changelog != nil
        pretext = "此版更新內容：\n#{changelog}"
    end
    changelog_update
    current_version = get_version_number(target: ENV["TARGET_NAME"])
    slack_message("Submit version #{current_version} to App review.", pretext, "product_manager", true)
end

lane :daily_archive do
    if ENVied.NOT_WORK_ON_CI
        next
    end
    
    if have_new_feature
        alpha
    else
        slack_message("Skip Daily Archive", "developer", true)
    end
end

private_lane :have_new_feature do
    last_archive_commit_hash = sh('git log -1 --grep "version\[build\]:" --format=%h | tr -d "\n"')
    new_faeture = sh("git log --oneline --grep 'feat:' --grep 'fix:' #{last_archive_commit_hash}...")
    new_faeture != "" ? true : false
end

def have_new_commit
    last_archive_commit_hash = sh('git log -1 --grep "version\[build\]:" --format=%h | tr -d "\n"')
    new_commit = sh("git log --oneline #{last_archive_commit_hash}...")
    new_commit != "" ? true : false
end

lane :install_library do
  pod_install if File.file?("../Podfile")
  carthage_install if File.file?("../Cartfile")
end

lane :pod_install do
    if is_ci
        last_archive_commit_hash = sh('git log -1 --grep "version\[build\]:" --format=%h | tr -d "\n"')
        pod_change = sh("git log --oneline --grep 'lib\[pod\]: update' #{last_archive_commit_hash}...")
        repo_update = pod_change != "" ? true : false
    else 
        repo_update = false
    end
    cocoapods(
        repo_update: repo_update
    )
end

lane :carthage_install do
  carthage(
      platform: "iOS",
      use_xcframeworks: true,
      cache_builds: true
  )
end

lane :carthage_update do |options|
    carthage(
        command: "update",
        dependencies: options[:library],
        platform: "iOS",
        use_xcframeworks: true,
        cache_builds: true
    )
end

def archive(scheme, method = "app-store")
    match(readonly: true) if is_ci unless ENVied.USE_AUTO_SIGN
    install_library if is_ci
    gym(
        scheme: scheme,
        export_method: method,
        cloned_source_packages_path: "Packages",
        silent: true
    )
end

lane :upload_api do |options|
    changelog = get_changelog

    if ENV["FASTLANE_LANE_NAME"] == "release"
        if !File.exists?("./metadata/zh-Hant/release_notes.txt")
            sh("mkdir -p ./metadata/zh-Hant/ && touch ./metadata/zh-Hant/release_notes.txt")
        end
        sh("echo \"#{changelog}\" > ./metadata/zh-Hant/release_notes.txt")

        deliver(
            submission_information: {
                add_id_info_uses_idfa: false
            },
            reject_if_possible: true, # Rejects the previously submitted build if it's in a state where it's possible
            submit_for_review: true,
            phased_release: true,
            skip_screenshots: true,
            precheck_include_in_app_purchases: false,
            force: true  # Skip the HTML report file verification
        )

        sh("git checkout ./metadata/zh-Hant/release_notes.txt")
    else
        distribute_method = options[:distribute_method]
        if distribute_method == "Firebase"
            setupEnv("FIREBASE_APP_ID", "firebase app id")
            setupEnv("FIREBASE_TESTERS", "firebase testers")

            firebase_app_distribution(
                app: ENV["FIREBASE_APP_ID"],
                release_notes: get_changelog,
                testers: ENV["FIREBASE_TESTERS"],
                debug: !is_ci
            )
        else
            setupEnv("EXTERNAL_GROUPS", "external groups", false)

            is_distribute_external = ENV["EXTERNAL_GROUPS"] ? true : false
            testflight(
                app_identifier: CredentialsManager::AppfileConfig.try_fetch_value(:app_identifier),
                changelog: changelog,
                distribute_external: is_distribute_external,
                groups: ENV["EXTERNAL_GROUPS"],
                notify_external_testers: is_distribute_external
            )
        end
    end
end

# Library

lane :update_library do
    update_bundle
    fastlane_update
    update_cocoapods
    update_carthage
    git_push
end

def update_bundle
    sh("bundle update")
    sh("bundle exec fastlane update_plugins")
    if is_git_status_dirty
        git_add(path: "./Gemfile.lock")
        sh("git commit -m 'lib[bundle]: update'")
    end
end

lane :fastlane_update do
    update_fastlane
    git_commit(path: "./Gemfile.lock", message: "lib[fastlane]: update")
end

def update_cocoapods
    sh(command: "bundle exec pod update")
    if is_git_status_dirty
        git_commit(path: "./Podfile.lock", message: "lib[pod]: update")
    end
end

def update_carthage
    carthage(
        command: "update",
        platform: "iOS",
        use_xcframeworks: true,
        cache_builds: true
    )
    if is_git_status_dirty
        git_commit(path: "./Cartfile.resolved", message: "lib[carthage]: update")
    end
end

# Change Log

def get_changelog
    if ENV["FASTLANE_LANE_NAME"] == "release"
        read_changelog(
            section_identifier: "[New Release]"
        )
    else
        read_changelog(
            section_identifier: "[New Build]"
        )
    end
end

def changelog_update
    if ENV["FASTLANE_LANE_NAME"] == "release"
        update_changelog(
            section_identifier: "[New Release]",
            updated_section_identifier: "Release #{get_version_number(target: ENV["TARGET_NAME"])}"
        )
    else
        update_changelog(
            section_identifier: "[New Build]",
            updated_section_identifier: "Build #{get_build_number}"
        )
    end

    if is_git_status_dirty
        git_commit(path: "./CHANGELOG.md", message: "docs[changelog]: update")
        git_push
    end
end

# Meta Data

desc "Take screenshots and upload."
lane :screenshots do |options|
    setupEnv("SNAPSHOT_BUNDLE_ID", "snapshot bundle id")
    setupEnv("SNAPSHOT_PATH", "snapshot path")
    setupEnv("BUNDLE_ID", "bundle id")

    match(
        app_identifier: ENV["SNAPSHOT_BUNDLE_ID"],
        type: "development",
        readonly: true
    ) if is_ci unless ENVied.USE_AUTO_SIGN
    install_library if is_ci

    if options[:devices] != []
        devices = options[:devices]
    end
    devices = ["iPhone 13 Pro Max", "iPhone 13", "iPhone 8 Plus"]
    snapshot(
        devices: devices,
        output_directory: ENV["SNAPSHOT_PATH"],
        override_status_bar: true,
        localize_simulator: true,
        reinstall_app: true,
        app_identifier: ENV["SNAPSHOT_BUNDLE_ID"],
        skip_open_summary: is_ci,
        skip_helper_version_check: is_ci
    )
    deliver(
        skip_binary_upload: true,
        run_precheck_before_submit: false,
        screenshots_path: ENV["SNAPSHOT_PATH"],
        force: is_ci, # Skip the HTML report file verification
        app_identifier: ENV["BUNDLE_ID"]
    )
    slack_message("螢幕截圖成功", "product_manager", true)
end

lane :update_snapshot do
    setupEnv("BUNDLE_ID", "bundle id")
    setupEnv("SNAPSHOT_PATH", "snapshot path")

    deliver(
        app_identifier: ENV["BUNDLE_ID"],
        screenshots_path: ENV["SNAPSHOT_PATH"],
        overwrite_screenshots: true,
        skip_binary_upload: true,
        run_precheck_before_submit: false
    )
end

lane :update_meta_data do
    deliver(
        skip_binary_upload: true,
        run_precheck_before_submit: false,
        metadata_path: "./metadata",
        force: is_ci # Skip the HTML report file verification
    )
end

lane :submit_for_review do
    deliver(
        submission_information: {
            add_id_info_uses_idfa: false,
            export_compliance_uses_encryption: false
        },
        reject_if_possible: true, # Rejects the previously submitted build if it's in a state where it's possible
        submit_for_review: true,
        automatic_release: false,
        phased_release: true,
        skip_binary_upload: true,
        skip_metadata: true,
        skip_screenshots: true,
        precheck_include_in_app_purchases: false,
        force: is_ci  # Skip the HTML report file verification
    )

    slack_message("iOS 最新版本送審成功", "product_manager", true)
end

# Others

desc "Register new device."
lane :add_device do
    fastlane_require "tty-prompt"
    prompt = TTY::Prompt.new

    device_name = prompt.ask("Device Name: ", required: true)
    device_id = prompt.ask("UDID: ", required: true)
    register_devices(
        devices: {
            device_name => device_id
        }
    )
    match(force: true)

    UI.success "[fastlane] Automatically add device, Name: #{device_name}, Identifier: #{device_id}."
end

lane :generate_badge_icon do
#   uncomment next line if not install imagemagick yet
#   sh("brew install imagemagick")
    fastlane_require "tty-prompt"
    setupEnv("PROJECT_NAME", "project name")

    prompt = TTY::Prompt.new
    icon = prompt.ask("icon:", required: true)
    image = prompt.ask("image:", required: false)
    if image == nil
        image = icon
    end

    add_badge(
        custom: "./fastlane/badge/#{image}.png",
        glob: "/**/#{ENV["PROJECT_NAME"]}/Assets.xcassets/AppIcon-#{icon}.appiconset/*.{png,PNG}"
    )
end

def git_push
    if is_ci
        push_to_git_remote(
            remote: "origin",
            local_branch: git_branch,
            remote_branch: git_branch
        )
    end
end

def is_git_status_dirty
  diff = sh("git diff")
  diff != ""
end

desc "commit README.md auto update after new fastfile run"
lane :readme_auto_update do
    if is_git_status_dirty
        git_commit(path: "./fastlane/README.md", message: "doc[ci]: fastlane README auto update")
        git_push
    end
end

# Error

error do |lane, exception|
    if is_ci
        slack_message("#{lane} failed (┛`д´)┛︵┴─┴", exception.respond_to?(:error_info) ? exception.error_info.to_s : exception.to_s, "developer", false)
    end
end

# Slack

desc "Send notification messaage."
lane :send_notification do |options|
    slack_message(options[:title], options[:message], options[:inform_level], options[:success])
end

def slack_message(title, message = nil, inform_level, success)
    setupEnv("SLACK_PRODUCTMANAGER_WEBHOOK_URL", "slack product manager webhook url")
    setupEnv("SLACK_DEVELOPER_WEBHOOK_URL", "slack developer webhook url")

    if ENV["DEBUG_MODE"] == "true"
      inform_level = "developer"
    end
    if inform_level == "product_manager"
        slack_webhook_url = ENV["SLACK_PRODUCTMANAGER_WEBHOOK_URL"]
    elsif inform_level == "developer"
        slack_webhook_url = ENV["SLACK_DEVELOPER_WEBHOOK_URL"]
    else
        slack_webhook_url = ENV["SLACK_DEVELOPER_WEBHOOK_URL"]
    end

    fields = []
    commit = last_git_commit
    fields = fields.push({ "title": "Lane", "value": ENV["FASTLANE_LANE_NAME"], "short": true }) if inform_level == "developer"
    fields = fields.push({ "title": "Branch", "value": git_branch, "short": true }) if inform_level == "developer"
    fields = fields.push({ "title": "Git Message", "value": commit[:message], "short": false }) if inform_level == "developer"
    fields = fields.push({ "title": "Git Author", "value": commit[:author], "short": true }) if inform_level == "developer"
    fields = fields.push({ "title": "Git Hash", "value": commit[:abbreviated_commit_hash], "short": true }) if inform_level == "developer"

    fields = fields.push({ "title": "Version", "value": get_version_number(target: ENV["TARGET_NAME"]), "short": true })
    fields = fields.push({ "title": "Build Number", "value": get_build_number, "short": true })

    build_by = is_ci ? ENV["CI_NAME"] : sh(command: "git config user.name")
    fields = fields.push({ "title": "Built by", "value": build_by }) if inform_level == "developer"

    fields = fields.push({ "title" => "Run Page", "value" => "https://github.com/#{ENV["GITHUB_REPOSITORY"]}/actions/runs/#{ENV["RUN_ID"]}" }) if is_ci && inform_level == "developer"

    if message
      pretext = "*#{title}*\n#{message}"
    else
      pretext = title
    end

    slack(
        pretext: pretext,
        success: success,
        default_payloads: [],
        attachment_properties: {
            fields: fields
        },
        slack_url: slack_webhook_url
    )
end
