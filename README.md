# TwinCAT 3 Event logger サンプルコード

本サンプルコードにて TwinCAT 3.1 build 4024 にて [Event logger](https://infosys.beckhoff.com/content/1033/tc3_eventlogger/index.html?id=8504177607767980219) を扱う実装例をご紹介します。以下の特徴を備えます。

* Visualization ( PLC HMI ) にアラームの一覧表示、確認ボタン操作、アラーム解除操作ができるようになっています。疑似的にSeverityがAlarm, Warning, Informationの3つを出力するボタンを付属しています。
* FB_TcArgument を通じて、状況に応じた付加的な2つの引数データをアラームテキストに結合しています。本サンプルコードでは、乱数で発生したエラーコードを表示させています。
* Visualization向けに用意された [イベントテーブル](https://infosys.beckhoff.com/content/1033/tc3_plc_intro/3524166155.html?id=4373836669159094324) へ表示させる最適な[ データ構造体 ST_ReadEvent ](https://infosys.beckhoff.com/content/1033/tcplclib_tc2_utilities/14563649035.html?id=1412304240424485687)へ出力する専用のファンクションブロック `FB_Tc3EventRead` を用意しました。この構造体は、TwinCAT2向けのEvent loggerに用意されたもので、TwinCAT3.1 build 4024ではこの型式に出力するネイティブファンクションブロックは用意されていません。
* `FB_Tc3EventRead` は `InterfaceEventViewer` インターフェースを実装したファンクションブロックとなっています。TwinCAT2向けの構造体`ST_ReadEvent` を出力するだけでなく、同インターフェースを実装することで他に様々な型式のビューを生成する事ができます。

<video src="./assets/visu_alarm.mp4">

## サンプルコード解説

### イベントクラスへの登録

[ユーザイベントの作成](https://beckhoff-jp.github.io/TwinCATHowTo/event_logger/make_user_event.html#)節をご覧いただき、イベントクラスとイベントを作成してください。本サンプルプロジェクトでは、`UserEventClass`で3つ登録されています。

### パラメータ設定

`GVLs` > `AlarmEventParm` にて、前節で作成したイベント数を設定します。

``` pascal
{attribute 'qualified_only'}
VAR_GLOBAL CONSTANT
    ALARM_MAX_COUNT: UINT := 3;
END_VAR
```

### グローバル変数の定義

`GVLs` > `GVL` に、それぞれのアラームの振る舞い仕様を設定します。

``` pascal
VAR_GLOBAL CONSTANT
    // Database ID
    TARGET_DBID : UINT := 1;
END_VAR
VAR_GLOBAL
    (*
    eSeverity ::
    TYPE TcEventSeverity : (
        Verbose  := 0, 
        Info     := 1, 
        Warning  := 2, 
        Error    := 3, 
        Critical := 4);
    END_TYPE
    *)
    AlarmDatabase : ARRAY [1..AlarmEventParam.ALARM_MAX_COUNT] OF ST_EventMetadata := [
        (    
            nEventId := 1,                // イベントID 1 
            eSeverity := 3,             // TcEventSeverity
            bWithConfirmation := TRUE,    // 新規アラームと確認済みアラームの区別が必要か否か
            bWithResetOperation := TRUE    // アラーム解除のリセット操作が必要か否か
        ),
        (    nEventId := 2, 
            eSeverity := 2, 
            bWithConfirmation := TRUE,
            bWithResetOperation := FALSE
        ),
        (    nEventId := 3, 
            eSeverity := 1, 
            bWithConfirmation := FALSE,
            bWithResetOperation := TRUE
        )
    ];
END_VAR
```

### メインプログラム定義

まず、PLC スタート後1サイクルだけ実行されるプログラムで、初期化を行います。

``` pascal
PROGRAM MAIN
VAR
    // Alarm calculation function block
    init : BOOL;
    fb_observer : FB_EventObserver;
    event_table_view : FB_Tc3EventRead;
    i : UINT;
END_VAR

(*
    FB_Observer内の FB_Alarm インスタンス配列に対して、GVL.AlarmDatabseで定義したイベントクラスの情報で紐付ける。
*)
IF NOT init THEN
    FOR i := 1 TO AlarmEventParam.ALARM_MAX_COUNT DO
        fb_observer.event_table.init_instance(
            eventClass := TC_EVENT_CLASSES.UserEventClass,
            nEventId := GVL.AlarmDatabase[i].nEventId,
            eSeverity := GVL.AlarmDatabase[i].eSeverity,
            bWithConfirmation := GVL.AlarmDatabase[i].bWithConfirmation,
            bWithResetOperation := GVL.AlarmDatabase[i].bWithResetOperation
        );
    END_FOR
    fb_observer.event_table.viewer := event_table_view;
    init := TRUE;
END_IF
```
また、個々のアラームの発報制御プログラムを記述します。

``` pascal
// アラーム1　（Alarmレベル） 発報制御
alarm_instance REF= fb_observer.event_table.get_alarm(1);            // 1番目のエラー配列に作成した FB_Alarm インスタンス参照を取り出す
IF __ISVALIDREF(alarm_instance) THEN        
    IF is_error THEN                                                // is_errorが エラー状態 bit
        error_code_random(Seed := 1);
        error_argument := TO_INT(error_code_random.Num * 32767.0);    // 疑似的に乱数によりエラーコードを生成
        alarm_instance.ipArguments.Clear();                            // Event Class に登録したEventのDisplay text の {0} {1} に展開した文字をクリアにする。
        tmp_string := 'Error Code';                                    // F_STRING() の引数は VAR_IN_OUT なのでリテラルは使えない。一旦仮変数にセットする。
        alarm_instance.add_arguments(F_STRING(tmp_string));            // Event Class に登録したEventのDisplay text の {0} 部分に埋め込まれる値。 T_Arg型でセット。
        alarm_instance.add_arguments(F_Int(error_argument));        // Event Class に登録したEventのDisplay text の {1} 部分に埋め込まれる値。 T_Arg型でセット。
    END_IF
    alarm_instance.set_activate := is_error;                        // エラー状態の通知。エラーテキストに　{0} や {1} などの付加的な情報が無ければこの 1 行だけで良い。
END_IF

// アラーム2　（Warningレベル） 発報制御
alarm_instance REF= fb_observer.event_table.get_alarm(2);
IF __ISVALIDREF(alarm_instance) THEN    
    IF is_warning THEN
        error_code_random(Seed := 1);
        error_argument := TO_INT(error_code_random.Num * 32767.0);
        alarm_instance.ipArguments.Clear();
        tmp_string := 'Warning Code';    
        alarm_instance.add_arguments(F_STRING(tmp_string));
        alarm_instance.add_arguments(F_Int(error_argument));
    END_IF
    alarm_instance.set_activate := is_warning;
END_IF


// アラーム3　（Informationレベル） 発報制御
alarm_instance REF= fb_observer.event_table.get_alarm(3);
IF __ISVALIDREF(alarm_instance) THEN
    IF is_info THEN
        error_code_random(Seed := 1);
        error_argument := TO_INT(error_code_random.Num * 32767.0);
        alarm_instance.ipArguments.Clear();
        tmp_string := 'Status';
        alarm_instance.add_arguments(F_STRING(tmp_string));
        alarm_instance.add_arguments(F_Int(error_argument));
    END_IF
    alarm_instance.set_activate := is_info;
END_IF
```
アラーム確認ボタンの受付と、アラームリセットボタンの受付のプログラムを定義します。

``` pascal
//　アラームリセットボタン受付
IF reset_button_input THEN
    fb_observer.event_table.try_clear_all_alarm();
END_IF

// アラーム確認ボタン受付
IF confirm_button_input THEN
    fb_observer.event_table.confirm_all_alarm();
END_IF
```

稼働しているIPCのロケール情報から、アラーム画面に表示させるタイムゾーンを設定します。

``` pascal
VAR
    fbGetTimeZoneInformation    : FB_GetTimeZoneInformation := (bExecute := TRUE);
END_VAR

// IPCのロケール設定の読み出しと、Visualization向けイベントテーブル表示FBへのタイムゾーン情報設定
IF fbGetTimeZoneInformation.bBusy THEN
    fbGetTimeZoneInformation.bExecute := FALSE;        
END_IF
fbGetTimeZoneInformation();
event_table_view.tzinfo := fbGetTimeZoneInformation.tzInfo;
```

常時実行タスクが全て定義されている `FB_EventObserver` を実行します。また、Visualizationからのコンボボックス設定により表示させるアラームの言語コードを切り替えます。

``` pascal
// オブザーバ（監視）オブジェクトの実行
CASE lang_select_options OF
    0: fb_observer.nLangId := 1033;
    1: fb_observer.nLangId := 1041;
END_CASE
fb_observer(event_class := TC_EVENT_CLASSES.UserEventClass);
```

最後に、Visualizationに表示されるボタンの点滅、点灯制御などを定義します。

``` pascal
// 表示用の変数制御　（アクティブアラーム数、未確認アラーム数、ボタン点滅制御、リセット可能状態）
active_count := fb_observer.event_table.active_event_count;
status_active := active_count > 0;
unconfirmed_count := fb_observer.event_table.unconfirmed_event_count;

status_resetable := fb_observer.event_table.set_event_count = 0 AND status_active;

IF unconfirmed_count > 0 THEN
    blink_timer(IN := NOT blink_timer.Q);
    IF blink_timer.Q THEN
        status_unconfirm := NOT status_unconfirm;
    END_IF
ELSE
    status_unconfirm := status_active;
END_IF
```

## Visualizationの所在

PLCプロジェクトの `VISUs` / `Visualization` が親画面になります。
