[![Build Status](https://travis-ci.org/mmcc007/todo.svg?branch=dev)](https://travis-ci.org/mmcc007/todo)
[![Build Status](https://travis-ci.org/mmcc007/todo.svg?branch=master)](https://travis-ci.org/mmcc007/todo)

#  CICD for Flutter

This is a slightly opinionated approach to CICD that seems to match well with flutter.

The idea is to do all development in a `dev` branch and when ready for beta, do a beta release
to both `Google Play Console` and `App Store Connect`. 

Each time a new beta is ready, a 'start beta' command is issued and a new beta is started 
and automatically released to testers.

Then when ready for general release, a 'release' command is issued and the `dev` branch is automatically 
merged with the `master` branch and the release is uploaded to the `Apple Store` and the 
Google `Play Store`.

In this way it is guaranteed that:
 1. An app build for iOS and Android always has the same version name
and build number. 
2. The build used to pass beta testing is the same build that is shipped to the stores. No rebuild
required.
3. The source of a build can always be traced back from the build number.

Information about the build can be displayed in an About section of the shipped app for support
and bug fixing.

# Implementation

The 'start beta' and 'release' commands mentioned above are implemented using a combination of a repository server, 
a build server and fastlane. 

1. Repository Server    
    The repository server can run any git server, such as GitHub, GitLab, etc. The git tag, in 
    semver format, is used as the verion name.
2. Build Server    
    The build server can be provided by Travis, Cirrus, an internal server running GitLab, Jenkins, etc.. The build server
    should provide a method to get the build number. The build number is used to ensure the release in
    both stores can be related back to the source code that was used to generate the app.    
3. Fastlane    
    Fastlane plays two roles: 
    1. To build and upload the ios and android app.    
        This occurs on the build server.
    1. To implement the `start_beta` and `release` command    
        This occurs on the local machine and triggers processes on the build server.
        
# Setup

There are a lot of setup steps to take to get this working. But keep in mind that this only has
to be done once. 

Many of these steps have to be taken anyway to release an app. So I figured, may as well gather
all these steps into one place and add some automation!

If you want to make releases on demand, it is well worth the effort!

## Application Setup

Decide on an application ID for your app that is unique in both stores. For example, `com.mycompany.todo`. This will be used
in several places to configure your app.

If you don't already have the latest (or near latest) version of the project set up, it is 
recommended that you build a new project and overlay your new project with your existing
project code. For example:

    flutter create --project-name todo --org com.mycompany todo
    cd <my project>
    cp -r lib test test_driver pubspec.yaml <location of new project>/todo

To enable CICD-managed revision control comment out the `version` in pubspec.yaml

    # version: 1.0.0+1

If you have already customized your icons:

    cd <my project>
    tar cf - android/app/src/main/res ios/Runner/Assets.xcassets | ( cd <location of new project>; tar xf -)

As with any mobile app, the following changes are required.
On android:

1. Update the application id in `android/app/build.gradle`:

    ````
    applicationId "com.mycompany.todo"
    ````
    
    `versionCode` and `versionName` can be ignored. These are updated automatically by the CICD.

On ios:

1. Open Xcode. For example:

    open ios/Runner.xcworkspace
    
1. Using XCode update the `Display Name` to the name the user will see.
2. Using XCode update the `Bundle Identifier` to the same as the application id used on android, eg, 'com.mycompany.todo'.

    `Version` and `Build` can be ignored. These are updated automatically by the CICD.
3. Disable automatic signing
4. In `Signing (Release)` select the provisioning profile create during match setup. 
    For example, use the following provisioning profile:
        
        match AppStore com.mycompany.todo
        
    Note: if match is not already set-up you will have to return to this step after match is set-up.

## Fastlane setup
Copy the fastlane files from this example to your app. For example

    cd <location of this app>
    tar cf - fastlane android/fastlane ios/fastlane script .gitignore | ( cd <localation of new project>; tar xf -)
    
and modify metadata to suit your needs.

Update the `package_name` in `ios/fastlane/Appfile` and `android/fastlane/Appfile` to your 
application ID. For example:
 
    package_name("com.mycompany.todo")


## Android App Store Connect setup
`App Store Connect` requires that the application be set-up before builds
can be uploaded automatically. Therefore, you should take the following steps:

### Create new app in store
    
1. Go to `App Store Connect` (https://play.google.com/apps/publish)
 
3. Click on `Create Application` and provide a title for your app. For example, `Todo`.

4. Provide additional required information `Short Description`, `Long Description`, screenshots, etc...

    For icon generation try https://iconsflow.com/, https://makeappicon.com/, https://pub.dartlang.org/packages/flutter_launcher_icons
    
    For auto screenshot generation see https://pub.dartlang.org/packages/screenshots
    
    For Feature Graphic generation see https://www.norio.be/android-feature-graphic-generator/

5. When all necessary information is provided, click on `Save Draft`. 

### Sign app
An android app requires signing. This is implemented using a private key that you generate yourself.
It is important that you manage this private key carefully. For example, never check it into your
repo.

However, to automate the build, the CICD needs access to this private key. 
This CICD expects to find a password protected encrypted version of the private key in the repo. The
password to unencrypt the private key is provided in the `KEY_PASSWORD` described above.

Follow the directions at https://developer.android.com/studio/publish/app-signing to learn about
app signing.
 
1. If you do not already have a keystore, generate a new keystore:


    keytool -genkey -v -keystore android/key.jks -keyalg RSA -keysize 2048 -validity 10000 -alias key
    keytool -importkeystore -srckeystore android/key.jks -destkeystore android/key.jks -deststoretype pkcs12
    rm android/key.jks.old
    
2. Create the `android/key.properties`:


    storePassword=<store password>
    keyPassword=<key password>
    keyAlias=key
    storeFile=../key.jks

Then encrypt them as follows:
1. Add the following to your `.gitignore`:


    **/android/key.properties
    **/android/key.jks
    
2. Encrypt both files with:
    
    
    openssl enc -aes-256-cbc -salt -in android/key.jks -out android/key.jks.enc -k $KEY_PASSWORD
    openssl enc -aes-256-cbc -salt -in android/key.properties -out android/key.properties.enc -k $KEY_PASSWORD

3. Enable android release builds in `android/app/build.gradle`:
    
    Replace:
    ````
    apply from: "$flutterRoot/packages/flutter_tools/gradle/flutter.gradle"
    
    android {
    ````
    with
    ````
    apply from: "$flutterRoot/packages/flutter_tools/gradle/flutter.gradle"
    
    def keystoreProperties = new Properties()
    def keystorePropertiesFile = rootProject.file('key.properties')
    if (keystorePropertiesFile.exists()) {
        keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
    }
    
    android {
    ````
    
    and replace:
    ````
        buildTypes {
            release {
                // TODO: Add your own signing config for the release build.
                // Signing with the debug keys for now, so `flutter run --release` works.
                signingConfig signingConfigs.debug
            }
            profile {
                matchingFallbacks = ['debug', 'release']
            }
        }
    ````
    with:
    ````
        signingConfigs {
            release {
                keyAlias keystoreProperties['keyAlias']
                keyPassword keystoreProperties['keyPassword']
                storeFile file(keystoreProperties['storeFile'])
                storePassword keystoreProperties['storePassword']
            }
        }
        buildTypes {
            release {
                signingConfig signingConfigs.release
            }
        }
    ````
3. Push key.jks.enc and key.properties.enc and android/app/build.gradle to the source repo.


### Upload first apk

Upload the first apk manually (this is required so `App Store Connect` knows the App ID)
1. Goto `App Releases` and open a beta track. Click on `Manage` and `Edit Release`
2. Click on `Continue` to allow Google to manage app signing key
3. Click on `Browse Files` to upload the current apk (built with `flutter build apk`) from `build/app/outputs/apk/release/app-release.apk`.

## App Store Connect setup
The equivalent steps for the android store have to be taken for the iOS store.

### Sign app
Signing is done via Fastlane match.

Configure a private match server from Fastlane.

This requires access to a private repository. You can use a private GitHub repository or
use a private repository from another repository provider, or provide your own.
    
For information on how to set-up a match private repo see: https://docs.fastlane.tools/actions/match

This CICD does not need the `Matchfile` created during match setup. However, it can be created
temporarily to run some match setup
1. Initialize match. It will ask for the location and write access to the remote private match repo.
    
    ````
    fastlane match init
    ````
    
2. Create your provisioning profile and app. You will have to pick a unique name for the app for 
end users

    ````
    fastlane produce -u user@email.com -a com.mycompany.todo
    ````
    
3. Sync the match repo with the app store

    ````
    fastlane match appstore
    ````
    
   This will create a provisioning profile that will be used during app setup.
4. Delete the Matchfile (as it contains secure info)

### Create required images
1. Icons

    Upload will fail if required icons are missing from the Asset Catalog. To generate a complete set
    of icons from a single image, see https://makeappicon.com. This will generate a complete Asset
    Catalog. Overwrite the existing catalog using:

        cp <location of downloaded icons>/ios/AppIcon.appiconset/* ios/Runner/Assets.xcassets/AppIcon.appiconset
        
2. Screenshots

    Screenshots must be included in upload. Screenshots can be generated automatically using (for
    both android and ios) using https://pub.dartlang.org/packages/screenshots.

1. App Store Icon

    iOS Apps must include a 1024x1024px App Store Icon in PNG format.
    
    See https://makeappicon.com/
    Store in `ios/fastlane/metadata/app_icon.png`
    
2. App Store Icon for iPad

    Since flutter supports iPad a related app icon is required of exactly '167x167' pixels, in .png format for iOS versions supporting iPad Pro
    
## Repo server setup
Assuming you have an empty remote repo:
1. Commit files on your local repo
2. Create a `dev` branch on your local repo

    git checkout -b dev

3. Push your local repo to the remote repo.

    git push --set-upstream origin dev

4. On the repo server, it is recommended to set the `master` branch to protected and `dev` as the default branch. This is to prevent accidental manual pushes to the `master` branch.

After this point
the remote `master` should be protected and should never be pushed-to manually. There should never
be a reason to even checkout the local `master` branch locally. All CICD git commands should be issued from
the local `dev` branch.

## Build server setup

If your Apple ID under your Apple Developer Account has 2-factor authentication enabled, 
you must create a new Apple ID without 2-factor authentication. This can be done using your
existing Apple Developer account. See https://appstoreconnect.apple.com/access/users.

Add the following secret variables to your preferred build server (Travis, or GitLab, etc... ):


    FASTLANE_USER
    FASTLANE_PASSWORD
    FASTLANE_SESSION
    GOOGLE_DEVELOPER_SERVICE_ACCOUNT_ACTOR_FASTLANE
    KEY_PASSWORD
    PUBLISHING_MATCH_CERTIFICATE_REPO
    MATCH_PASSWORD
    
   * FASTLANE_USER
    
        This is your login name to the Apple Developer username. For example, user@email.com.
    
   * FASTLANE_PASSWORD
    
        This is your Apple Developer password. For travis, if there are special characters the 
        password should be enclosed in single quotes.
        
   * FASTLANE_SESSION
    
        You need to generate a login session for your CI machine for your Apple Developer account in advance. You can do this by 
        running:
        
            fastlane spaceauth -u user@email.com
            
        For details, see https://github.com/fastlane/fastlane/tree/master/spaceship#2-step-verification
            
   * GOOGLE_DEVELOPER_SERVICE_ACCOUNT_ACTOR_FASTLANE
    
        This is required to login to `Google Play Console`. This is a private key. It should be
        surround with single quotes to be accepted by Travis. It can be generated on 
        https://console.developers.google.com
        
   * KEY_PASSWORD
    
        This is the password to the encrypted app private key stored in `android/key.jks.enc` and
        the related encrypted properties files stored in `android/key.properties.enc`
        
   * PUBLISHING_MATCH_CERTIFICATE_REPO
    
        This is the location of the private match repo. For example, https://private.mycompany.com/private_repos/match
     
   * MATCH_PASSWORD
   
        The password used while setting up match.
        
## Local repo setup
Add an initial semver tag locally as your first version name:

    git tag 0.0.0
    
# Usage

## Starting a beta

To start a beta

Make sure you are in the dev directory in root of repo (and all files are committed and uploaded to remote)

    fastlane start_beta


This will push the committed code in the local `dev` to the remote `dev`, increment the semver version number and generate a tag and trigger a build of the app 
for ios and android and release the beta to testers 
automatically on both stores.

When ready to start a new beta
simply re-issue the command.

Semver can be incremented using:

    fastlane start_beta patch (the default)
    fastlane start_beta minor
    fastlane start_beta major
    
## Release to both stores

To release to both stores, from root of repo enter:

    fastlane release


This will confirm that the local `dev` is committed locally and as a precaution it confirms that no
push is required from the local `dev` to the remote `dev`. Then it will merge the remote `dev` to 
the remote `master`. This will 
trigger the build server to promote the build used in beta testing to a release in both stores. 
The remote `master` now contains the most current code. A rebuild of the beta-tested build is not required.

# Todo example

This todo app was taken from https://github.com/brianegan/flutter_architecture_samples/tree/master/example/vanilla

Permission pending!
