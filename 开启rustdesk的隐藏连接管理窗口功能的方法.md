```markdown
# 开启 RustDesk 的隐藏连接管理窗口功能的方法

## 1. main.dart 中修改

```dart
void runConnectionManagerScreen() async {
  await initEnv(kAppTypeConnectionManager);
  _runApp(
    '',
    const DesktopServerPage(),
    MyTheme.currentThemeMode(),
  );
  final hide = await bind.cmGetConfig(name: "hide_cm") == 'true';
  gFFI.serverModel.hideCm = hide;
  gFFI.serverModel.notifyListeners(); // 强制通知UI刷新
  if (hide) {
    await hideCmWindow(isStartup: true);
  } else {
    await showCmWindow(isStartup: true);
  }
  setResizable(false);
  // Start the uni links handler and redirect links to Native, not for Flutter.
  listenUniLinks(handleByFlutter: false);
}
```

## 2. desktop_setting_page.dart 中修改

```dart
Widget hide_cm(bool enabled) {
    return ChangeNotifierProvider.value(
        value: gFFI.serverModel,
        child: Consumer<ServerModel>(builder: (context, model, child) {
          final enableHideCm = model.approveMode == 'password' &&
              model.verificationMethod == kUsePermanentPassword;
          onHideCmChanged(bool? b) {
            if (b != null) {
              // 1. 更新底层配置
              bind.mainSetOption(key: 'allow-hide-cm', value: bool2option('allow-hide-cm', b));
              // 2. 保存「当前是否隐藏复选框」的状态（新增逻辑）
              bind.mainSetOption( key: 'hide_cm', value: bool2option('hide_cm', b));
              // 2. 同步更新模型状态（关键：修复勾选状态不更新问题） 
              model.hideCm = b; 
              // 3. 通过模型的现有逻辑触发窗口操作
              model.notifyListeners();
              // 4. 立即执行窗口显示/隐藏
              if (b) {hideCmWindow(); } 
              else {showCmWindow(); }
              // 4. 通知UI更新 
              model.notifyListeners(); 
            }
          }
```

## 3. desktop_setting_page.dart 中修改

取消掉下边两行带没带注释：
```dart
        if (usePassword)
               hide_cm(!locked).marginOnly(left: _kContentHSubMargin - 6),
```

## 4. server_model.dart 中修改（重构 updatePasswordModel 实现状态加载）

```dart
updatePasswordModel() async {
    var update = false;
    final temporaryPassword = await bind.mainGetTemporaryPassword();
    final verificationMethod =
        await bind.mainGetOption(key: kOptionVerificationMethod);
    final temporaryPasswordLength =
        await bind.mainGetOption(key: "temporary-password-length");
    final approveMode = await bind.mainGetOption(key: kOptionApproveMode);
    final numericOneTimePassword =
        await mainGetBoolOption(kOptionAllowNumericOneTimePassword);
    
    // 从配置文件加载 hide_cm 状态
    final hideCmOption = await bind.mainGetOption(key: 'hide_cm');
    var newHideCmState = hideCmOption == 'Y' || hideCmOption == 'true';
    if (!(approveMode == 'password' &&
        verificationMethod == kUsePermanentPassword)) {
      newHideCmState = false;
    }

    if (_approveMode != approveMode) {
      _approveMode = approveMode;
      update = true;
    }
    var stopped = await mainGetBoolOption(kOptionStopService);
    final oldPwdText = _serverPasswd.text;
    if (stopped ||
        verificationMethod == kUsePermanentPassword ||
        _approveMode == 'click') {
      _serverPasswd.text = '-';
    } else {
      if (_serverPasswd.text != temporaryPassword &&
          temporaryPassword.isNotEmpty) {
        _serverPasswd.text = temporaryPassword;
      }
    }
    if (oldPwdText != _serverPasswd.text) {
      update = true;
    }
    if (_verificationMethod != verificationMethod) {
      _verificationMethod = verificationMethod;
      update = true;
    }
    if (_temporaryPasswordLength != temporaryPasswordLength) {
      if (_temporaryPasswordLength.isNotEmpty) {
        bind.mainUpdateTemporaryPassword();
      }
      _temporaryPasswordLength = temporaryPasswordLength;
      update = true;
    }
    if (_allowNumericOneTimePassword != numericOneTimePassword) {
      _allowNumericOneTimePassword = numericOneTimePassword;
      update = true;
    }
    
    // 当 hideCm 状态改变时，更新UI
    if (this.hideCm != newHideCmState) {
      this.hideCm = newHideCmState;
      if (desktopType == DesktopType.cm) {
        if (this.hideCm) {
          await hideCmWindow();
        } else {
          await showCmWindow();
        }
      }
      update = true;
    }
    
    if (update) {
      notifyListeners();
    }
  }
```
```