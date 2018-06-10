fastlane_version "1.81.0"

default_platform :ios



platform :ios do
  before_all do

    setup_jenkins
    
    if is_ci?
        puts "Jenkins build"
        ENV['SLACK_URL']            = ENV['SLACK_FASTLANE_TOKEN']
        ENV['GYM_OUTPUT_DIRECTORY'] = ENV['BUILD_OUTPUT_PATH']
    else
        ENV['SLACK_URL']            = "https://hooks.slack.com/services/..."
        ENV['GYM_OUTPUT_DIRECTORY'] = 'output'
    end
  end


  ### public lanes ###
  
  desc "create bundle ID with push notification, pem file, and provisioning profile"
  lane :setup_enterprise do

      setup_info

      team_id   = "xxxxxxxxxx"
      sku       = "0123456789"
      user_name = "appleAcount@email.com"
      
      ENV['PRODUCE_USERNAME'] = user_name
      ENV['PRODUCE_TEAM_ID']  = team_id
      ENV['PRODUCE_SKU']      = sku
      
      ENV['PEM_USERNAME'] = ENV['PRODUCE_USERNAME']
      ENV['PEM_TEAM_ID']  = ENV['PRODUCE_TEAM_ID']
      
      ENV['SIGH_TEAM_ID']  = team_id
      ENV['SIGH_USERNAME'] = user_name
      ENV['SIGH_APP_IDENTIFIER'] = ENV['bundle_id']
      
      ## app ID and push notification service ##
      produce(app_identifier: ENV['bundle_id'],
              app_name: ENV['app_name'],
              skip_itc: true,
              skip_devcenter: true,
              language: "English"
      )
      
      sh "produce enable_services --push-notification --app_identifier " + ENV['bundle_id']
      
      pem(app_identifier: ENV['bundle_id'])
      
      #sigh(cert_id: "7A35123BC5FF8666139CB378DCEE095F80CAC330") # does not work
      sh "sigh -u " + user_name + " -a " + ENV['bundle_id']
  end

  lane :build do
      
      # configure env: GYM_OUTPUT_NAME, GYM_SCHEME, PROJECT_PATH, GYM_WORKSPACE (optional)
      configureEnv
      
      ENV['BUILD_NUMBER']   = increment_build_number(build_number: ENV['BUILD_NUMBER'])
      ENV['VERSION_NUMBER'] = get_version_number(xcodeproj: ENV['PROJECT_PATH'])
      puts "version: " + ENV['VERSION_NUMBER']
      
      ENV['CHANGELOG'] = changelog_from_git_commits(include_merges: false,
                                                    tag_match_pattern: "jenkins/*/#{git_branch}/*")
      
      if is_ci?
        # switch lane
        if git_branch.include? 'master'
            production
        elsif git_branch.include? 'release'
            staging
        else
            development
        end
      end
      
      if build_configuration.eql? "Production"
      elsif build_configuration.eql? "Staging"
      elsif build_configuration.eql? "Debug"
      end
      
      before_actions
      compile
      after_actions

      if enable_snapshot
          # https://github.com/fastlane/fastlane/tree/master/snapshot
          snapshot(scheme: ENV['GYM_SCHEME'])
      end
  end


  ### override lanes ###

  lane :setup_info do
    # implement this lane locally using override_lane
    # required: ENV['bundle_id'] and ENV['app_name']
  end

  lane :configureEnv do
    # implement this lane locally using override_lane
  end

  lane :enable_snapshot do 
    # return false
    false
  end

  lane :build_configuration do
    # return "Release"
    "Release"
  end

  lane :compile do
    gym
    # gym(use_legacy_build_api: true) # this was option removed with Xcode 8.3
  end

  lane :before_actions do 
    # implement this lane locally using override_lane
  end

  lane :after_actions do 
    # implement this lane locally using override_lane
  end

  lane :deploy do
    crashlytics
  end
  
  
  ### private lanes ###
  
  private_lane :development do
      puts "develop branch"
      
      ENV['CHANGELOG'] = changelog_from_git_commits
  end
  
  private_lane :staging do
      puts "production branch"
      ENV['VERSION_NUMBER'] = increment_version_number(bump_type: "patch")
      
      commit_version_bump(
              message: 'Update staging version',
              xcodeproj: ENV['PROJECT_PATH'],
              force: true)
      add_git_tag(
          grouping: 'staging',
          prefix: 'v',
          build_number: ENV['VERSION_NUMBER'])
  end
  
  private_lane :production do
      puts "master branch"
      
      # add_git_tag(
      #     grouping: 'production',
      #     prefix: 'v',
      #     build_number: ENV['VERSION_NUMBER']
      # )
  end
  
  
  private_lane :push_changes_to_master do |options|
      puts "push changes to master"
      sh("git push")
#      push_to_git_remote
  end

  private_lane :push_changes_to_production do |options|
      puts "push changes to production"
      sh("git push")
  end

  private_lane :export do |options|
      ## export environment variables
      
      bundle_id      = CredentialsManager::AppfileConfig.try_fetch_value(:app_identifier)
      version_number = ENV['VERSION_NUMBER']
      build_number   = ENV['BUILD_NUMBER']
      
      changelogs = ENV['CHANGELOG']                         # lane_context[SharedValues::FL_CHANGELOG]
      changelog  = ''
      unless changelogs.nil?
        changelog  = changelogs.split(/\n+/).join('\\ \n')
      end
      
      output_name       = ENV['GYM_OUTPUT_NAME'] + '.ipa'   # lane_context[SharedValues::IPA_OUTPUT_PATH]
      output_directory  = ENV['GYM_OUTPUT_DIRECTORY']
      output_file_path  = output_directory + '/' + output_name
      
      open(options[:file_path], 'a') { |f|
          
          f.puts "BRANCH_NAME=#{git_branch}"
          f.puts "BUILD_NANE=#{output_name}"
          f.puts "BUILD_OUTPUT_PATH=#{output_directory}"
          
          f.puts "BUILD_SUMMARY=Jenkins build"
          f.puts "RD_NOTES=#{changelog}"
          
          # required
          f.puts "BUILD_BUNDLE_ID=#{bundle_id}"
          f.puts "BUILD_NUMBER=#{build_number}"
          f.puts "BUILD_VERSION=#{version_number}"
          f.puts "BUILD_OUTPUT_FILENAME=#{output_name}"
      }
  end


  ### after & error lanes ###
  
  after_all do |lane|
    
    if is_ci?
      if git_branch.include? 'master'
          push_changes_to_master
      elsif git_branch.include? 'production'
          push_changes_to_production
      end
      
      export(file_path: "../properties/postbuild.properties")
    
    end
    
    slack(
      message: "Successfully built an iOS app.",
      payload: {            # optional, lets you specify any number of your own Slack attachments
        'Bundle ID' => CredentialsManager::AppfileConfig.try_fetch_value(:app_identifier),
        '.ipa file' => ENV['GYM_OUTPUT_NAME'] + '.ipa',
        'Version' => ENV['VERSION_NUMBER'],
        'Build number' => ENV['BUILD_NUMBER'],
        'Scheme' => ENV['GYM_SCHEME'],
        'Git branch' => git_branch
      },
      default_payloads: [],
    )
  end

  error do |lane, exception|
      slack(
        message: exception.message,
        success: false
      )
  end
end
