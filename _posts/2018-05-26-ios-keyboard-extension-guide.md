---
title:  "iOS Custom Keyboard Guide"
date:   2018-05-26 00:00:00
description: Small overview of common problems and issues that arise when using UIStackView.
categories:
- iOS Custom Keyboard
permalink: ios-custom-keyboard-guide
comments: true
---

Starting with iOS 8, Apple introduced iOS App Extensions which includes Custom Keyboard functionality. From that moment developers are able to create their own implementation of keyboards which users can use instead of system one. This the win for both sides.

The API which Apple provides to build custom keyboard is not big and it’s easy to learn it quickly, but there are many caveats you will meet to build really user-friendly one. Therefore in this article, you will find additional information on specific topics which I hope will speed up your custom keyboard development process.

I assume that you already know the basics of iOS and Objective-C (or Swift) programming language. The knowledge of iOS Keyboard extension development is preferred but not mandatory. However, if you’re not familiar with it the best place to start is [official documentation](https://developer.apple.com/library/content/documentation/General/Conceptual/ExtensibilityPG/CustomKeyboard.html). Also, the provided solutions work in iOS 10 and above. Some things may work and can be true for older versions as well. 

Please note that solutions will not be rejected by Apple review team because they are already included in my App Store shipped application. Therefore do not hesitate to include these code recipes to your project.

{% include toc %}

## Understanding the permissions

Many iOS developers including me wrongly understand what does the `RequestsOpenAccess` flag do in **Info.plist** file.

As the documentation says:
> RequestsOpenAccess — This Boolean value, NO by default, expresses whether a custom keyboard wants to enlarge its sandbox beyond that needed for a basic keyboard.

In other words, by setting the value to `YES` you will be able to use the below capabilities:
* Access to Location Services, the Address Book database, and the Camera Roll
* Option to use a shared container with the keyboard containing app
* Play sound on key press
* Make network requests
* Use UIPasteboard
* Access to iCloud, Game Center and In-App Purchase
* Work with managed apps

Right? Yes and no. Some of these capabilities can be accessed after you set this flag to `YES` and others will still need a user to enable this by hand.

Specifically, after enabling `RequestsOpenAccess` flag you will get access to these features for free:
* Play sound on key press
* Partially use a shared container. You will be able to read and write from containing app but the extension will be able only to read.

Other features can be accessed only if the user enabled the **Allow Full Access** flag in iOS device settings for your keyboard.

## How to know if a user allowed a full access?

After reading the above paragraph you may ask then how to check if a user allowed a full access?

Well, there is a main tricky method to check for a full access. It tries to call `UIPasteboard` class methods which are available only if user allowed full access to your keyboard. If methods work then your extension has a full access and opposite otherwise.

Note that starting from iOS 11 there is a variable called `hasFullAccess` in `UIInputViewController` instances which is already returning the needed result. However, for previous iOS versions, you need to implement that function by yourself.

Here is a sample implementation in Objective-C:
``` objc
- (BOOL)openAccessIsEnabled {
    BOOL hasFullAccess = false;
    if (@available(iOS 11.0, *)) {
        hasFullAccess = [self hasFullAccess];
    } else {
        if (@available(iOS 10.0, *)) {
            UIPasteboard *pasty = [UIPasteboard generalPasteboard];
            NSString *previousString = pasty.string;
            pasty.string = @"TEST";
            if (pasty.hasStrings) {
                hasFullAccess = true;
                pasty.string = previousString;
            }
        } else if ([UIPasteboard generalPasteboard]) {
            hasFullAccess = true;
        }
    }

    return hasFullAccess;
}
```

Swift version:
``` swift
override var hasFullAccess: Bool {
    if #available(iOS 11.0, *) {
        return super.hasFullAccess
    }
    if #available(iOS 10.0, *) {
        let original: String? = UIPasteboard.general.string
        UIPasteboard.general.string = " "
        let val: Bool = UIPasteboard.general.hasStrings
        if let str = original {
            UIPasteboard.general.string = str
        }
        return val
    }
    return UIPasteboard.general.isKind(of: UIPasteboard.self)
}
```

Another note is that that this method works only from extension side. Unfortunately, I didn’t find the way to check full access from the containing app. If you have the better solution you can share it with others in comments :)

Also, if you’re checking the full access very often in iOS 10 and older versions then it’s good to create a variable which will contain the cached version of above method.

## Is keyboard on?

Next popular question is to know from containing app if your iOS Keyboard extension is enabled in Settings. There is no official way to do this. However, there is a method proposed in [Stack Overflow](https://stackoverflow.com/) which works like a charm.

Here it is (Objective-C version):
``` objc
- (BOOL)isKeyboardEnabled {
    NSArray *keyboards = [[NSUserDefaults standardUserDefaults] objectForKey:@"AppleKeyboards"];
    NSUInteger index = [keyboards indexOfObject:@"YOUR_KEYBOARD_EXTENSION_BUNDLE_ID"];

    return index != NSNotFound;
}
```

Swift version:
``` swift
func isKeyboardEnabled() -> Bool {
    guard let keyboards = UserDefaults.standard.object(forKey: "AppleKeyboards") as? [String] else {
        return false
    }
    return keyboards.contains("YOUR_KEYBOARD_EXTENSION_BUNDLE_ID")
}
```

As you may guess the **AppleKeyboards** key in standard `NSUserDefaults` will return array of bundle ids of all enabled keyboards. The only thing to do there is to check if the list contains your keyboard extension bundle id.

## Is keyboard currently active?

As you may see the iOS Keyboard extension is very limited. There is no direct way to communicate with your containing app and the extension. You may create an App Group and save your app settings in `NSUserDefaults` like so:
``` objc
NSUserDefaults *userDefaults = [[NSUserDefaults alloc] initWithSuiteName:@"YOUR_APP_GROUP_ID"];
[userDefaults setObject:@"SOME_VALUE" forKey:@"SOME_KEY"];
[userDefaults synchronize];
```

But if full access is not given it will be one-way communication. Specifically, from you containing app to the keyboard extension.

To overcome the problem, you may use deep linking to send events from extension to your containing app. Use this method carefully because it’ll open and switch to containing app if keyboard extension currently is shown in another app.

For example, if you need to know when your keyboard extension will be visible/activated in order to show a welcome message or do other action you can do the following:
1. Create a custom URL scheme, i.e with identifier **mykeyboard**.
2. To catch the link call implement the `- (BOOL)application:(UIApplication *)app openURL:(NSURL *)url options:(NSDictionary<UIApplicationOpenURLOptionsKey, id> *)options;` method in containing app AppDelegate class:
    
    ``` objc
    - (BOOL)application:(UIApplication *)app openURL:(NSURL *)url options:(NSDictionary<UIApplicationOpenURLOptionsKey,id> *)options {
        if ([url.absoluteString isEqualToString:@"SET_KEYBOARD_ACTIVATED"]) {
            [[NSNotificationCenter defaultCenter] postNotificationName:@"KEYBOARD_ACTIVATED_NOTIFICATION" object:nil];
            return true;
        }
        return false;
    }
    ```

    Swift version:
    ``` swift
    func application(_ app: UIApplication, open url: URL, options: [UIApplicationOpenURLOptionsKey : Any] = [:]) -> Bool {
        if url.absoluteString == "SET_KEYBOARD_ACTIVATED" {
            NotificationCenter.default.post(name: NSNotification.Name(rawValue: "KEYBOARD_ACTIVATED_NOTIFICATION"), object: nil)
            return true;
        }
        return false;
    }
    ```
3. Get shared `UIApplication` from keyboard extension and call the needed URL.

    ``` objc
    NSUserDefaults *userDefaults = [[NSUserDefaults alloc] initWithSuiteName:@"YOUR_APP_GROUP_ID"];

    if (![userDefaults boolForKey:@"IS_KEYBOARD_ACTIVATED"]) {
        UIResponder *responder = self;
        NSURL *url = [NSURL URLWithString:@"SET_KEYBOARD_ACTIVATED"];
        while ((responder = [responder nextResponder]) != nil) {
            if ([responder respondsToSelector:@selector(openURL:)]) {
                [responder performSelector:@selector(openURL:) withObject:url];
            }
        }
    }
    ```
    Swift version:
    ``` swift
    let userDefaults = UserDefaults(suiteName: "YOUR_APP_GROUP_ID")
    let url = URL(string: "SET_KEYBOARD_ACTIVATED")

    if let isKeyboardActivated = userDefaults?.bool(forKey: "IS_KEYBOARD_ACTIVATED"),
        isKeyboardActivated,
        let aUrl = url {
        var responder: UIResponder? = self
        while responder != nil {
            if let application = responder as? UIApplication {
                application.perform("openURL:", with: aUrl)
            }
            responder = responder?.next
        }
    }
    ```

In above steps, we assume that:
* The **SET_KEYBOARD_ACTIVATED** string value is your URL scheme. For example, `mykeyboard://set_keyboard_activated`.
* **KEYBOARD_ACTIVATED_NOTIFICATION** is the name of notification you want to post.
* **YOUR_APP_GROUP_ID** the unique identifier of your created App Group. For example, `group.com.myapp.Keyboard`.
* **IS_KEYBOARD_ACTIVATED** in this sample is just a key in `NSUserDefaults` to know if the welcome message was shown before or not.

## Open Settings

By knowing that some of the functionality in your iOS Keyboard extension will need a full access you may be interested in a way how to easily navigate users to your keyboard preferences in iOS Settings app.

Before asking for a full access the best practice is to explain why you need it and guarantee or show that you don’t use users keystrokes to get private data from them.
Assuming you’ve already done it, let me show you the way how to open the Settings app and navigate to your keyboard preferences.

For iOS 11 and above we will use the **UIApplicationOpenSettingsURLString** key to open your app preferences in the Settings app. However, for below iOS versions, there is no way to directly open your app preferences in the Settings app. Instead, we can open Keyboard settings where a user can add your keyboard and allow full access. Do do this, first of all, you need to add a couple of additional URL schemes with identifiers **App-Prefs** and **prefs**. The **App-Prefs** will be used for iOS 10 and in other hand **prefs** will be used for iOS 9 and below. 

Then we can start to implement the method which will navigate us to your keyboard settings:
``` objc
- (void)openAppSettings {
    if (@available(iOS 11.0, *)) {
        [[UIApplication sharedApplication] openURL:[NSURL URLWithString:UIApplicationOpenSettingsURLString] options:@{} completionHandler:nil];
    } else if (@available(iOS 10.0, *)) {
        [[UIApplication sharedApplication] openURL:[NSURL URLWithString:@"App-Prefs:root=General&path=Keyboard/KEYBOARDS"] options:@{} completionHandler:nil];
    } else {
        [[UIApplication sharedApplication] openURL:[NSURL URLWithString:@"prefs:root=General&path=Keyboard/KEYBOARDS"] options:@{} completionHandler:nil];
    }
}
```

Swift version:
``` swift
func openAppSettings() {
    var responder: UIResponder? = self
    var sharedApplication: UIResponder?
    while responder != nil {
        if let application = responder as? UIApplication {
            sharedApplication = application
            break
        }
        responder = responder?.next
    }

    guard let application = sharedApplication else { return }

    if #available(iOS 11.0, *) {
        application.perform("openURL:", with: URL(string: UIApplicationOpenSettingsURLString))
    } else {
        if #available(iOS 10.0, *) {
            application.perform("openURL:", with: URL(string: "App-Prefs:root=General&path=Keyboard/KEYBOARDS"))
        } else {
            application.perform("openURL:", with: URL(string: "prefs:root=General&path=Keyboard/KEYBOARDS"))
        }
    }
}
```

As you can see for iOS 10 and below the difference is very small. The path idea is in `root=General&path=Keyboard/KEYBOARDS`. The prefix like **App-Prefs** or **prefs** will open the Settings app. Then it will navigate to **General** section of Settings app. After it will open **Keyboard** preferences and in the end show us **Keyboards** settings. In that window, you will see a list of all enabled keyboards. A user will be able to add your keyboard to the list or by pressing on your keyboard option see the allow full access option to enable it.

The difference for iOS 11 is that you will be navigated directly to your app settings. The user will need then to press on **Keyboards** option in that window and then enable full access. There is a small detail. While testing sometimes your app settings not shown in the main Settings app list. To fix this you may need to close Settings app or in the worst scenario restart you simulator or the device. I didn’t encounter this problem in production.

## Playing sound

As the official documentation says you can play key press sound using method of the `UIDevice` `playInputClick`. However, you are not limited. The `playIntputClick` plays only the sound of character keys. For other types of keys like space, delete, or shift you may want to use sounds like in default keyboard.

First of all, let’s add **AudioToolbox** framework, import it and then assume that you have these key types:
``` objc
typedef enum KeyType: NSInteger {
    KeyTypeDefault,
    KeyTypeCapsLock,
    KeyTypeDelete,
    KeyTypeExtraSymbolsSwitch,
    KeyTypeSymbolsSwitch,
    KeyTypeLettersSwitch,
    KeyTypeNextKeyboard,
    KeyTypeEmojis,
    KeyTypeSpace,
    KeyTypePunctuations,
    KeyTypeReturn
} KeyType;
```

Swift version:
``` swift
enum KeyType: Int {
    case defaultKey,
    capsLock,
    delete,
    extraSymbolsSwitch,
    symbolsSwitch,
    lettersSwitch,
    nextKeyboard,
    emojis,
    space,
    punctuations,
    returnKey
}
```

Then to play sounds like in the default keyboard you may implement function like this:
``` objc
- (void)playSound:(KeyType)keyType {
    if (soundIsOn) {
        SystemSoundID clickPress = 1123;
        SystemSoundID deletePress = 1155;
        SystemSoundID modifierPress = 1156;

        SystemSoundID current = clickPress;

        if (keyType == KeyTypeDelete) {
            current = deletePress;
        } else if (keyType != KeyTypeDefault) {
            current = modifierPress;
        }

        // fallback for iOS 9 and below
        if (@available(iOS 10.0, *)) {
        } else {
            current = 1104;
        }
       dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
            AudioServicesPlaySystemSound(current);
        });
    }
}
```

Swift version:
``` swift
func playSound(_ keyType: KeyType) {
    if soundIsOn {
        let clickPress: SystemSoundID = 1123;
        let deletePress: SystemSoundID = 1155;
        let modifierPress: SystemSoundID = 1156;

        var current: SystemSoundID = clickPress;

        if keyType == .delete {
            current = deletePress
        } else if keyType != .defaultKey {
            current = modifierPress
        }

        // fallback for iOS 9 and below
        if #available(iOS 10.0, *) {
        } else {
            current = 1104;
        }

        DispatchQueue.global().async {
            AudioServicesPlaySystemSound(current)
        }
    }
}
```

The **SystemSoundID** ids like **1123**, **1155**, and **1156** are present for iOS 10 and above. For older versions you may use id **1104**.

## Handing auto capitalization, text field and return key types

The best way to handle the auto capitalization and the change of text field or return key type is in `UITextInputDelegate` delegate method called `-(void)textDidChange:(id<UITextInput>)textInput` in your `UIInputViewController` instance class.

For example, like this:
``` objc
- (void)textDidChange:(id<UITextInput>)textInput {
    if (self.textDocumentProxy && self.currentKeyboard.keyboardType != self.textDocumentProxy.keyboardType) {
        [self setKeyboardType];
    }
    if (self.textDocumentProxy && self.currentKeyboard.returnKeyType != self.textDocumentProxy.returnKeyType) {
        [self setReturnKeyType];
    }
    [self setCapsIfNeeded];
}
```

Swift version:
``` swift
override func textDidChange(_ textInput: UITextInput?) {
    if textDocumentProxy && currentKeyboard.keyboardType != textDocumentProxy.keyboardType {
        setKeyboardType()
    }
    if textDocumentProxy && currentKeyboard.returnKeyType != textDocumentProxy.returnKeyType {
        setReturnKeyType()
    }
    setCapsIfNeeded()
}
```

In above sample code, the **currentKeyboard** is just a custom class that wraps keyboard logic and contains variables called **keyboardType** and **returnKeyType** which are instances of `UIKeyboardType` and `UIReturnKeyType` respectivetly. Methods like `setKeyboardType` and `setReturnKeyType` can be implemented differently. In best scenario they should change the interface of your keyboard properly regarding the text field and return key types.

In `setCapsIfNeeded` method you will need to do these three steps:
1. Check the `UITextAutocapitalizationType` property like this `self.textDocumentProxy.autocapitalizationType`
2. Get text before cursor like this `self.textDocumentProxy.documentContextBeforeInput`
3. Decide to use lower or upper case

Note, that you also need to set keyboard and return key types, and set correct capitalization in the beginning when you initialize the keyboard. For example, in `- (void)viewDidLoad` method.

## Conclusion

I hope after reading this article you now know more information how to build your custom iOS Keyboard extension or have an idea how to solve your problems.

To build fully featured keyboard you will need also add functionality like auto correction, word suggestions, one-handed swipe typing and more. But these topics are big on their own and need a separate article.

To guide a little bit more I suggest also to explore `UILexicon` and `UITextChecker` classes API if you want to build auto correction and suggestion functionality with the iOS internal tools. But note, that these APIs are limited and in most scenarios you will need to implement auto correction and suggestion functionality by yourself.

Here are some useful links:
* [Official documentation](https://developer.apple.com/library/content/documentation/General/Conceptual/ExtensibilityPG/CustomKeyboard.html)
* [Tasty Imitation Keyboard](https://github.com/archagon/tasty-imitation-keyboard) - the custom iOS Keyboard extension which tries to imitate the default system keyboard.
