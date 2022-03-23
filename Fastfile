fastlane_require "tty-prompt"

default_platform(:ios)

ENV["FASTLANE_DONT_STORE_PASSWORD"] = "1"

before_all do |lane, options|
    if options[:fake_lane] != nil
        ENV["FASTLANE_LANE_NAME"] = options[:fake_lane]
    end
    setup_ci
    ensure_env_vars(
      env_vars: ['APP_STORE_CONNECT_API_KEY_ID', 'APP_STORE_CONNECT_API_ISSUER_ID', 'APP_STORE_CONNECT_API_P8_FILEPATH']
    )
    app_store_connect_api_key(
      key_id: ENV["APP_STORE_CONNECT_API_KEY_ID"],
      issuer_id: ENV["APP_STORE_CONNECT_API_ISSUER_ID"],
      key_filepath: ENV["APP_STORE_CONNECT_API_P8_FILEPATH"]
    )
end

desc "Build the project."
lane :build do
    match(readonly: true) if is_ci
    install_dependency if is_ci
    gym(
        scheme: ENV["SCHEME_DEV"],
        skip_archive: true
    )
    slack_message("Build Successfully", "developer", true)
end

desc "Run unit test."
lane :unit_test do |options|
    install_dependency
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
    ensure_env_vars(
      env_vars: ['TARGET_NAME', 'BUNDLE_ID', 'PROJECT_NAME']
    )

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
        testflight_build_number = latest_testflight_build_number(
            app_identifier: app_identifier_alpha,
            version: version,
            initial_build_number: 0
        )
        if testflight_build_number > build_number then
            build_number = testflight_build_number.to_i
        end

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

        build_number = build_number + 1
    end

    increment_build_number({
        build_number: build_number
    })
    
    if is_ci then
        sh(command: "git config --global user.name #{ENV["CI_NAME"]}")
        sh(command: "git config --global user.email #{ENV["CI_GIT_USER_EMAIL"]}")
    end
    commit_version_bump(
        message: "version[build]: #{get_build_number}",
        xcodeproj: "./#{ENV["PROJECT_NAME"]}.xcodeproj"
    )

    git_push
end

desc "Bump version with interactive command line mode."
lane :bump_version do |options|
    if options[:version]
        increment_version_number(
            version_number: options[:version]
        )
        increment_build_number(
            build_number: "1"
        )
        commit_version_bump(
            message: "version: #{(get_version_number(target: ENV["TARGET_NAME"]))}",
            xcodeproj: "./#{ENV["PROJECT_NAME"]}.xcodeproj"
        )
    else
        prompt = TTY::Prompt.new
        type = prompt.select("Which type of version to bump?", cycle: true) do |menu|
              menu.choice 'major'
              menu.choice 'minor'
              menu.choice 'patch'
        end

        increment_version_number(
            bump_type: type
        )
        increment_build_number(
            build_number: "1"
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

lane :beta do
    if have_new_commit
        bump_build_number
    end
    archive("Beta")
    upload_api
    changelog_update
    slack_message("TestFlight 上有新的 beta 版本", "product_manager", true)
end

lane :release do
    if have_new_commit
        bump_build_number
    end
    archive("Release")
    upload_api
    changelog = get_changelog
    changelog_update
    current_version = get_version_number(target: ENV["TARGET_NAME"])
    slack_message("Submit version #{current_version} to App review.", "此版本新增的功能\n#{changelog}", "product_manager", true)
end

lane :daily_archive do
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
    last_archive_commit_hash = sh('git log -1 --grep "version" --format=%h | tr -d "\n"')
    new_commit = sh("git log --oneline #{last_archive_commit_hash}...")
    new_commit != "" ? true : false
end

lane :install_dependency do
  pod_install if File.file?("../Podfile")
  carthage_install if File.file?("../Cartfile")
end

lane :pod_install do
  cocoapods(
      repo_update: is_ci
  )
end

lane :carthage_install do
  carthage(
      platform: "iOS",
      use_xcframeworks: true,
      cache_builds: true
  )
end

def archive(scheme)
  match(readonly: true) if is_ci
  install_dependency if is_ci
  if (ENV["FASTLANE_LANE_NAME"] != "release") && (ENV["ADD_BADGE_ICON"] == "true")
    if ENV["FASTLANE_LANE_NAME"] == "daily_archive"
      image = "alpha"
    else
      image = ENV["FASTLANE_LANE_NAME"]
    end
    generate_badge_icon(
      image: image,
      project_name: ENV["PROJECT_NAME"]
    )
  end
  gym(
    scheme: scheme,
    export_method: "app-store",
    cloned_source_packages_path: "Packages",
    silent: true
  )
  if (ENV["FASTLANE_LANE_NAME"] != "release") && (ENV["ADD_BADGE_ICON"] == "true")
    sh("git checkout ../#{ENV["PROJECT_NAME"]}/Assets.xcassets/")
  end
end

lane :upload_api do |options|
    changelog = get_changelog

    if ENV["FASTLANE_LANE_NAME"] == "release"
        sh("echo \"#{changelog}\" > ./metadata/zh-Hant/release_notes.txt")

        upload_to_app_store(
            submission_information: {
                add_id_info_uses_idfa: false
            },
            reject_if_possible: true, # Rejects the previously submitted build if it's in a state where it's possible
            submit_for_review: true,
            phased_release: true,
            skip_screenshots: true,
            force: true  # Skip the HTML report file verification
        )

        sh("git checkout ./metadata/zh-Hant/release_notes.txt")
    else
      is_distribute_external = ENV["EXTERNAL_GROUPS"] ? true : false
      testflight(
        app_identifier: CredentialsManager::AppfileConfig.try_fetch_value(:app_identifier),
        changelog: changelog,
        distribute_external: is_distribute_external,
        groups: ENV["AUTO_DISTRIBUTE_EXTERNAL_GROUPS"]
      )
    end
end

# Library

lane :update_dependency do
    update_bundle
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

def update_cocoapods
    sh(command: "bundle exec pod update")
    if is_git_status_dirty
        git_add(path: "./Podfile.lock")
        sh("git commit -m 'lib[cocoapods]: update'")
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
        git_add(path: "./Cartfile.resolved")
        sh("git commit -m 'lib[carthage]: update'")
    end
end

# Change Log

def get_changelog
    if ENV["FASTLANE_LANE_NAME"] == "release"
        read_changelog(
            changelog_path: ENV["CHANGELOG_PATH"],
            section_identifier: "[New Release]"
        )
    else
        read_changelog(
            changelog_path: ENV["CHANGELOG_PATH"],
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
        git_add(path: ENV["CHANGELOG_PATH"])
        sh("git commit -m 'docs[changelog]: update'")
        git_push
    end
end

# Meta Data

desc "Take screenshots and upload."
lane :screenshots do |options|
    match(
        app_identifier: ENV["SNAPSHOT_BUNDLE_ID"],
        type: "development",
        readonly: true
    ) if is_ci
    install_dependency if is_ci
    snapshot(
        devices: options[:devices],
        output_directory: ENV["SNAPSHOT_PATH"],
        skip_open_summary: is_ci,
        skip_helper_version_check: is_ci
    )
    upload_to_app_store(
        skip_binary_upload: true,
        run_precheck_before_submit: false,
        screenshots_path: ENV["SNAPSHOT_PATH"],
        force: is_ci, # Skip the HTML report file verification
        app_identifier: ENV["BUNDLE_ID"]
    )
end

lane :update_meta_data do
    upload_to_app_store(
        skip_binary_upload: true,
        run_precheck_before_submit: false,
        metadata_path: "./metadata",
				automatic_release: false,
        force: is_ci # Skip the HTML report file verification
    )
end

# Others

desc "Register new device."
lane :add_device do
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

lane :generate_badge_icon do |options|
  sh("brew install imagemagick")
  add_badge(
    custom: "./fastlane/badge/#{options[:image]}.png",
    glob: "/**/#{options[:project_name]}/Assets.xcassets/*.appiconset/*.{png,PNG}"
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
  slack_message("#{lane} failed (┛`д´)┛︵┴─┴", exception.respond_to?(:error_info) ? exception.error_info.to_s : exception.to_s, "developer", false)
end

# Slack

desc "Send notification messaage."
lane :send_notification do |options|
    slack_message(options[:title], options[:message], options[:inform_level], options[:success])
end

def slack_message(title, message = nil, inform_level, success)
  ensure_env_vars(
    env_vars: ['SLACK_PRODUCTMANAGER_WEBHOOK_URL', 'SLACK_DEVELOPER_WEBHOOK_URL']
  )

    if ENV["DEBUG_MODE"] == "true"
      inform_level = "developer"
    end
    if is_ci || ENV["HAVE_NO_CI"] == "true"
        if inform_level == "product_manager"
            slack_webhook_url = ENV["SLACK_PRODUCTMANAGER_WEBHOOK_URL"]
        elsif inform_level == "developer"
            slack_webhook_url = ENV["SLACK_DEVELOPER_WEBHOOK_URL"]
        else
            slack_webhook_url = ENV["SLACK_DEVELOPER_WEBHOOK_URL"]
        end
    else
        slack_webhook_url = ENV["SLACK_TEST_WEBHOOK_URL"]
    end

    fields = []
    commit = last_git_commit
    if inform_level == "developer"
      fields = fields.push({ "title": "Lane", "value": ENV["FASTLANE_LANE_NAME"], "short": true })
      fields = fields.push({ "title": "Branch", "value": git_branch, "short": true })
    end
    fields = fields.push({ "title": "Version", "value": get_version_number(target: ENV["TARGET_NAME"]), "short": true })
    fields = fields.push({ "title": "Build Number", "value": get_build_number, "short": true })
    if inform_level == "developer"
      fields = fields.push({ "title": "Git Message", "value": commit[:message], "short": true })
      fields = fields.push({ "title": "Git Author", "value": commit[:author], "short": true })
      fields = fields.push({ "title": "Git Hash", "value": commit[:abbreviated_commit_hash], "short": true })
      fields = fields.push({ "title": "Built by", "value": ENV["CI_NAME"] || sh(command: "git config user.name") })
    end

    if is_ci && inform_level == "developer"
      fields = fields.push({ "title" => "Run Page", "value" => "https://github.com/#{ENV["GITHUB_REPOSITORY"]}/actions/runs/#{ENV["RUN_ID"]}" })
    end

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
