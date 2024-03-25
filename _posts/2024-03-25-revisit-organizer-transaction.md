---
layout: post
title: '重读OrganizerTransaction'
published: false
---

## Prelude

[Organizer Transaction](https://github.com/michalbednarski/OrganizerTransaction) 是 Android 之神 Michał Bednarski 在两年前写的 Writeup, 主要解释了他在 Android 12L 中 利用 `WindowOrganizerController` 中错误的使用了 `Binder.clearCallingIdentity` 之后的状态获取 callingUid, 并用于设置 ActivityStarter 的对应属性, 从而导致可以以 `SYSTEM_UID`(1000) 打开任意 Activity 漏洞

彼时刚涉足安卓安全领域,并不能完全理解其中的利用的细节和精妙之处,同时原文中缺少一些背景信息交代, 如 `OrganizerTransaction` 相关 API 在 Android 开发时的使用场景等. 而最近我在看`androidx.window.extensions.jar`, 刚好重温一次神文, 所以续貂一下, 顺便增强自己的理解

## WindowManager Extensions

### ActivityEmbedding & 大屏适配

### Task & TaskFragment

### AIDL under the hood

## LaunchAnyWhere

### android.permission.START_ANY_ACTIVITY

```xml
    <!-- Allows an application to start any activity, regardless of permission
         protection or exported state.
         @hide -->
    <permission android:name="android.permission.START_ANY_ACTIVITY"
        android:protectionLevel="signature" />
```

### Binder

### grantUri

## canEmbeddedActivity

### relinquishTaskIdentity

### ChooserActivity

#### Selector

### ResolverActivity

## SurfaceControl
