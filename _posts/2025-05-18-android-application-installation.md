---
layout: post
title: 'Android application installation under the hood'
published: false
date: 2025-05-16
---

In my previous post, [Samsung’s ISVP: Betraying the Trust of Security Researchers](./samsung-bugbounty-scam), I shared my experience uncovering a vulnerability in Samsung mobile devices that enabled silent app installation. This flaw allowed non-privileged apps to bypass the [`android.permission.INSTALL_PACKAGES`](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/res/AndroidManifest.xml;drc=82ada6503a81af7eeed2924a2d2d942375f6c8c2;l=6054) signature permission, permitting app installations without user consent.
Despite Samsung’s ISVP policy specifying a $50,000 reward for such vulnerabilities, I received only $500, and my follow-up inquiries were ignored. While some recommended publicly disclosing the vulnerability details, doing so would breach the program’s terms of service.
Instead, this post examines the general Android app installation process, drawing on publicly available code from AOSP android-15.0.0_r1. The discussion is not specific to Samsung devices or the vulnerability I reported, focusing solely on open-source mechanisms.

## Key Components in Android Application Installation

This section outlines the primary components involved in Android’s application installation process, The discussion focuses on system services, privileged apps, and client interactions, providing a general overview without referencing specific vendor implementations.

### PackageInstallerService

The `PackageInstallerService` is a core system service that manages application installation sessions. Hosted in [PackageInstallerService.java](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/services/core/java/com/android/server/pm/PackageInstallerService.java), it provides methods to create, manage, and finalize installation sessions. Key methods include:

- **`createSession(sessionParam, installerPackageName, callingAttributionTag, userId): Int`**

  > Create a new session using the given parameters, returning a unique ID that represents the session. Once created, the session can be opened multiple times across multiple device boots.
  > The system may automatically destroy sessions that have not been finalized (either committed or abandoned) within a reasonable period of time, typically on the order of a day.

- **`openSession(sessionId): IPackageInstallerSession`**

  > Open an existing session to actively perform work. To succeed, the caller must be the owner of the install session.

- **`getSessionInfo(sessionId): PackageInstaller.SessionInfo`**

  > Return details for a specific session. Callers need to either declare <queries> element with the specific package name in the app's manifest, have the `android.permission.QUERY_ALL_PACKAGES`, or be the session owner to retrieve these details.

- **`setPermissionsResult(sessionId, accepted)`**

  > A hidden API for privileged callers to notify `PackageInstallerService` that the user has approved an installation request, allowing the installation to proceed.

### PackageInstallerSession

The `PackageInstallerSession` class handles the active phase of an installation session, enabling data writing and session finalization. Key methods include:

- **`openWrite(name, offset, length): ParcelFileDescriptor`**

  > Open a stream to write an APK file into the session.
  > The returned stream will start writing data at the requested offset in the underlying file, which can be used to resume a partially written file. If a valid file length is specified, the system will preallocate the underlying disk space to optimize placement on disk. It's strongly recommended to provide a valid file length when known.
  > You can write data into the returned stream, optionally call fsync(OutputStream) as needed to ensure bytes have been persisted to disk, and then close when finished. All streams must be closed before calling commit(IntentSender).

- **`commit(statusReceiver, forTransferred)`**

  > Commit the session when all constraints are satisfied. This is a convenient method to combine waitForInstallConstraints(List, PackageInstaller. InstallConstraints, IntentSender, long) and PackageInstaller. Session. commit(IntentSender).
  > Once this method is called, the session is sealed and no additional mutations may be performed on the session. In the case of timeout, you may commit the session again using this method or PackageInstaller. Session. commit(IntentSender) for retries.## PackageInstaller App

### PackageInstaller

The `PackageInstaller` system app (`com.android.packageinstaller`) facilitates sideloading of applications. It interacts with `PackageInstallerService` & `PackageInstallerSession` to provide a user interface for installation prompts and manage the installation process.

### OEM Privileged App Stores

While not directly covered in this post, OEM app stores (e.g., Google Play, Samsung Galaxy Store, Huawei AppGallery, Oppo/Vivo/Xiaomi stores) often have the `INSTALL_PACKAGES` permission. These apps are common entry points for installing third-party applications but are not central to the AOSP installation process discussed here.

### Client Applications

Client apps are non-privileged applications that initiate app installations. They interact with `PackageInstallerService` and (or) `PackageInstaller` through APIs or ACTIONs to request installation sessions

## Sideloading

Android supports two methods for requesting application installation: `PIA` and `Session Install`. These are informal terms used within the AOSP codebase. `PIA` corresponds to the `action android.intent.action.INSTALL_PACKAGE`, while `Session Install` is a newer API set introduced in Android L. `PIA` delegates most tasks to the `PackageInstaller` app, whereas `Session Install` offers apps finer control over the installation process. Additionally, only `Session Install` supports multi-package installations, sometimes called `splits`.

Since `PackageInstaller` internally uses `Session Install` to interact with `PackageInstallerSession` for installations triggered by `PIA`, I’ll discuss `Session Install` first.

### Session Install

The core APIs for `Session Install` were covered in the previous chapter. In essence, the process can be summarized as follows:

```kotlin
with(packageManager.packageInstaller) {
    val session = openSession(createSession(PackageInstaller.SessionParams(MODE_FULL_INSTALL)))
    session.openWrite("foobar.apk", 0, -1).use { to ->
        assets.open("magisk.apk").use { from ->
            from.copyTo(to)
            session.fsync(to)
        }
    }
    session.commit(IntentSender(object : IIntentSender.Stub() {
        override fun send(code: Int, intent: Intent, resolvedType: String?, whitelistToken: IBinder?, finishedReceiver: IIntentReceiver?, requiredPermission: String?, options: Bundle?) {
            intent.extras?.getParcelable(Intent.EXTRA_INTENT, Intent::class.java)?.let(::startActivity)
        }
    }))
}
```

This code creates a session, writes the apk content, and commits the session. Once commit is called, the PackageInstallerSession workflow begins.

- PackageInstallerSession dispatch `MSG_ON_SESSION_SEALED`, `MSG_STREAM_VALIDATE_AND_COMMIT`, `MSG_INSTALL` sequentially, one after the previous is done, 
when [`handleInstall`](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/services/core/java/com/android/server/pm/PackageInstallerSession.java;l=2845) is called (the callback of message `MSG_INSTALL`) the installation is suspended if the session for the installation needs user confirmation which is calculated in [`computeUserActionRequirement`](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/services/core/java/com/android/server/pm/PackageInstallerSession.java;l=1035) the result 



=======
## Sideloading

there are two approaches to request application install in android, `PIA` and `Session-based Installation`, the name is not formal but actively used in the AOSP code base,
`PIA` refers to **action `android.intent.action.INSTALL_PACKAGE`** while Session Install is a new set of API introduced since Android L, `PIA` delegates almost all work to PackageInstaller app, while with `Session Install`, app have better fine-grained tolerate to better control the whole process, also only `Session Install` can initiate multi-package installation, in some case referred as `splits`

### Session Install

Starting from Android L, google introduced the Session Installation API for sideloading, we'll first cover it here because under the hood, the traditional PIA installation

### PIA (`android.intent.action.INSTALL_PACKAGE`)

while being the most common & convenient way to request application install, it delegate most of the work to PackageInstaller, the overall process involves

- client start intent with action `android.intent.action.INSTALL_PACKAGE`, while having its data point to a content uri
- PackageInstaller receiver the intent and handle it in `StartInstall` Activity
  ```xml
  <activity android:name=".InstallStart"
              android:exported="true"
              android:excludeFromRecents="true">
          <intent-filter android:priority="1">
              <action android:name="android.intent.action.VIEW" />
              <action android:name="android.intent.action.INSTALL_PACKAGE" />
              <category android:name="android.intent.category.DEFAULT" />
              <data android:scheme="content" />
              <data android:mimeType="application/vnd.android.package-archive" />
          </intent-filter>
  </activity>
  ```
  ## while google is refactoring the process and ui guarded by flag [pia_v2](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/content/pm/flags.aconfig;drc=82ada6503a81af7eeed2924a2d2d942375f6c8c2;l=109), it is not feasible at current, we'll focus the traditional process here

### - Installation

==== below notes ===
``

- Session Installation
  - openSession
  - copy
  - commit

## Exploit
