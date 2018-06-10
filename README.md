### Import Fastfile 
```
  import_from_git(url: 'https://github.com/Jios/fastlane_files.git',
                  path: 'Fastfile')
```

### Example (iOS)
```
# import Fastfile from git 
import_from_git(url: 'https://github.com/Jios/fastlane_files.git',
                path: 'Fastfile')

default_platform :ios

platform :ios do
    
  # required lane
  override_lane :configureEnv do
    # gym, default configuration: 'Release'
    ENV['GYM_OUTPUT_NAME'] = "AppName"
    ENV['GYM_SCHEME']      = "AppName"
    ENV['GYM_WORKSPACE']   = "AppName.xcworkspace"
    ENV['PROJECT_PATH']    = "AppName.xcodeproj"

    # crashlytics
    ENV["CRASHLYTICS_API_TOKEN"]    = "..."
    ENV["CRASHLYTICS_BUILD_SECRET"] = "..."

  end


  # optional lanes #

  override_lane :enable_snapshot do
    # true/false
    false
  end
  
  override_lane :build_configuration do
    # "Release"/"Debug"
    "Release"
  end

  override_lane :before_actions do 
    # addtional actions before gym: sigh
    puts "No before gym/build actions"
  end

  override_lane :compile do
    # default: gym(use_legacy_build_api: true)
  end

  override_lane :after_actions do
    # addtional actions after gym (build)
    
    # make sure to select a provisioning profile for target
    crashlytics(
      groups: ['...'],
      emails: ['email@email.com'],
      notifications: true,
      debug: false,
      notes: "..."
    )
  end
end
```

### Upload to Crashlytics via command
```
# upload .ipa file
$ fastlane run Crashlytics ipa_path:path/to/.ipa \
                           api_token:... \
                           build_secret:... \
                           notifications:true \
                           debug:false \
                           groups:pms,rds \
                           emails:email1@email.com,email2@email.com

# upload .dSYM file
$ fastlane run upload_symbols_to_crashlytics dsym_path:./path/todSYM.zip \
                                             api_token:... \
                                             platform:ios
```

### Links
  * https://docs.fastlane.tools/actions/Actions/
  * https://docs.fastlane.tools/Advanced/
