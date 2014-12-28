---
title: 动态修改Android应用图标
date: 2014-12-28 13:17
tags: [android]
permalink: /2014/support-multiple-icons-in-android/
---
[Chainfire](www.chainfire.eu)(SuperSU作者)发现了一种可以[动态修改图标](www.chainfire.eu/articles/133/_TUT_Supporting_multiple_icons_in_your_app)的方法，挺有意思的，记录一下。

简单地说，该方法是使用 `<activity-alias>` 和 `PackageManager.setComponentEnabledSetting()` 来做到的。

## Android Manifest
注意`<activity>`标签中没有设置`launcher category`，另外，`<activity-alias>`中只有一个设置为`android:enabled=true`。

    :::xml linenums="True"
    <activity
        android:name=".MainActivity"
        android:label="@string/app_name">
        
    <intent-filter>
        <action android:name="android.intent.acton.MAIN" />
    </intent-filter>
    </activity>

    <activity-alias
        android:enabled="true"
        android:name=".MainActivity-default"
        android:label="@string/app_name"
        android:icon="@drawable/ic_launcher_default"
        android:targetActivity=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
    </activity-alias>

    <activity-alias
        android:enabled="false"
        android:name=".MainActivity-icon1"
        android:label="@string/app_name"
        android:icon="@drawable/ic_launcher_1"
        android:targetActivity=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
    </activity-alias>

## Java Code

    :::java
    private void setIcon(int iconId){
        Context ctx = getActivity();
        PackageManager pm = ctx.getPackageManager();
        ActivityManager am = ctx.getSystemService(Activity.ACTIVITY_SERVICE); 
        
        // enable/disable activity-aliases component
        pm.setComponentEnabledSetting(
            new ComponentName(ctx, "com.example.MainActivity-default"),
            iconId == ICON_DEFAULT?
                PackageManager.COMPONENT_ENABLED_STATE_ENABLED:
                PackageManager.COMPONENT_ENABLED_STATE_DISABLED,
            PackageManager.DONT_KILL_APP);

        pm.setComponentEnabledSetting(
            new ComponentName(ctx, "com.example.MainActivity-icon1"),
            iconId == ICON_1?
                PackageManager.COMPONENT_ENABLED_STATE_ENABLED:
                PackageManager.COMPONENT_ENABLED_STATE_DISABLED,
            PackageManager.DONT_KILL_APP);
        
        // Find launcher and kill it
        Intent i = new Intent(Intent.ACTION_MAIN);
        i.addCategory(Intent.CATEGORY_HOME);
        i.addCategory(Intent.CATEGORY_DEFAULT);
        List<ResolveInfo> resolves = pm.queryIntentActivities(i,0);
        for(ResovleInfo res : resolves){
            if(res.activityInfo != null){
                am.killBackgroundProcesses(res.acitivityInfo.packageName);
            }
        } 


        // Change ActionBar icon
        getActivity().getActionBar().setIcon(iconId);
    }


