
require "tty-prompt"

default_platform(:ios)

desc "Build the project."
lane :build do
    match
    install_library
	xcodebuild(
    	scheme: ENV["SCHEME_DEV"]
	)
    slack_message("Build Successfully", true)
end

desc "Run unit test."
lane :unit_test do |options|
    install_library
    scan(
        scheme: ENV["SCHEME_TEST"],
        device: "iPhone 8",
        test_without_building: options[:without_build]
    )
    slack_message("Unit Test Success", true)
end

desc "Bump build number."
lane :bump_build_number do |options|
    if is_ci then
        sh(command: "git config --global user.name #{ENV["CI_NAME"]}")
        sh(command: "git config --global user.email #{ENV["DEVELOPER_EMAIL"]}")
    end

    if options[:force]

    else
        if have_new_commit == false
            next
        end
    end

    version = get_version_number
    build_number = get_build_number.to_i

    testflight_build_number = latest_testflight_build_number(
        app_identifier: "#{ENV["BUNDLE_ID"]}.alpha",
        version: version,
        initial_build_number: 0
    )
    if testflight_build_number > build_number then
        build_number = testflight_build_number.to_i
    end

    appstore_build_number = app_store_build_number(
        live: false,
        version: version,
        app_identifier: ENV["BUNDLE_ID"],
        initial_build_number: 0
    )
    if appstore_build_number > build_number then
        build_number = appstore_build_number
    end

    increment_build_number({
        build_number: build_number + 1
    })
    commit_version_bump(
        message: "version[daily]: #{get_build_number}",
        xcodeproj: "./#{ENV["PROJECT_NAME"]}.xcodeproj"
    )

    git_push(false)
end

desc "Push a new alpha build to TestFlight"
lane :alpha do
    archive("Alpha")
    upload_api
    changelog_update
    slack_message("✈️ Successfully Deliver to TestFlight! (ﾉ>ω<)ﾉ ✈️", true)
end

desc "Daily Archive"
lane :daily_archive do
    if have_new_feature
        bump_build_number
        alpha
    else
        slack_message("Skip daily archive to TestFlight.", true)
    end
end

desc "Push a new alpha and beta build to TestFlight"
lane :beta do
    archive("Beta")
    upload_api
    changelog_update
    slack_message("✈️ Successfully Deliver to TestFlight! (ﾉ>ω<)ﾉ ✈️", true)
end

desc "Push a new alpha and release build to TestFlight"
lane :release do
    archive("Release")
    upload_api
    changelog_update
    slack_message("✈️ Successfully Deliver to TestFlight! (ﾉ>ω<)ﾉ ✈️", true)
end

desc "Take screenshots and upload."
lane :screenshots do |options|
    match(
        app_identifier: ENV["SNAPSHOT_BUNDLE_ID"],
        type: "development"
    )
    install_library
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

desc "Update meta data."
lane :update_meta_data do
    upload_to_app_store(
        skip_binary_upload: true,
        run_precheck_before_submit: false,
        metadata_path: "./metadata",
				automatic_release: false,
        force: is_ci # Skip the HTML report file verification
    )
end

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

desc "Bump version with interactive command line mode."
lane :bump_version do |options|
    prompt = TTY::Prompt.new
    type = prompt.select("Which type of version to bump?", cycle: true) do |menu|
          menu.choice 'major'
          menu.choice 'minor'
          menu.choice 'patch'
    end

    if options[:tag]
        add_git_tag(
            tag: "#{get_version_number}"
        )
    end

    increment_version_number(
        bump_type: type
    )
    increment_build_number(
        build_number: "0"
    )
    commit_version_bump(
        message: "version[#{type}]: #{get_version_number}",
        xcodeproj: "./#{ENV["PROJECT_NAME"]}.xcodeproj"
    )
end

lane :install_library do
#	 carthage(
#	 	 platform: "iOS",
#		 cache_builds: true
#    )
	sh(command: "../carthage.sh bootstrap --platform ios --cache-builds")
    cocoapods
end

private_lane :have_new_commit do
    log = sh(command: "git log --oneline -1 --format=%B | head -1")
    log.split.first != "version[daily]:" ? true : false
end

private_lane :have_new_feature do
    last_archive_commit = sh('git log -1 --grep "version\[daily\]:" --format=%h | tr -d "\n"')
    new_faeture = sh("git log --oneline --grep 'feat:' --grep 'fix:' #{last_archive_commit}...")
    new_faeture != "" ? true : false
end

def archive(scheme)
    match(readonly: true)
    install_library
    build_app(
        scheme: scheme,
        export_method: "app-store"
    )
end

lane :upload_api do |options|
    changelog = get_changelog

    lane = ENV["FASTLANE_LANE_NAME"]
    if lane == "release"
        sh("echo \"#{changelog}\" > ./metadata/zh-Hant/release_notes.txt")

        upload_to_app_store(
            submission_information: {
                add_id_info_uses_idfa: false
            },
            reject_if_possible: true, # Rejects the previously submitted build if it's in a state where it's possible
            submit_for_review: true,
            phased_release: true,
            skip_screenshots: true,
            force: is_ci  # Skip the HTML report file verification
        )
    else
        testflight(
            app_identifier: CredentialsManager::AppfileConfig.try_fetch_value(:app_identifier),
            changelog: changelog,
            distribute_external: options[:distribute_external],
            groups: options[:external_groups]
        )
    end
end

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
            updated_section_identifier: "Release #{get_version_number}"
        )
    else
        update_changelog(
            section_identifier: "[New Build]",
            updated_section_identifier: "Build #{get_build_number}"
        )
    end

    commit = last_git_commit
    if commit[:message].split.first == "version[daily]:"
        git_add(path: "#{ENV["CHANGELOG_PATH"]}")
        sh("git commit --amend --no-edit -m \"#{commit[:message]}\"")
        git_push(true)
    end
end

def git_push(force)
    if is_ci
        push_to_git_remote(
            remote: "origin",
            local_branch: git_branch,
            remote_branch: git_branch,
            force: force
        )
    end
end

error do |lane, exception, options|
    slack_message("Something Wrong! \n#{exception.error_info}", false)
end

def slack_message(message, success)
    commit = last_git_commit

    slack_webhook_url = is_ci ? ENV["SLACK_WEBHOOK_URL"] : ENV["SLACK_TEST_WEBHOOK_URL"]

    slack(
        pretext: message,
        success: success,
        payload: {
            "Lane" => ENV["FASTLANE_LANE_NAME"],
            "Commit Message" => commit[:message],
            "Commit Hash" => commit[:abbreviated_commit_hash],
            "Built by" => ENV["CI_NAME"] || "Anonymous"
        },
        default_payloads: [:git_branch, :git_author],
        slack_url: slack_webhook_url
    )
end
