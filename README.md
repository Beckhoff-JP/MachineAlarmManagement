# TwinCAT 3 Event logger サンプルコード

本サンプルコードにて TwinCAT 3.1 build 4024 にて [Event logger](https://infosys.beckhoff.com/content/1033/tc3_eventlogger/index.html?id=8504177607767980219) を扱う実装例をご紹介します。以下の特徴を備えます。

* Visualization ( PLC HMI ) にアラームの一覧表示、確認ボタン操作、アラーム解除操作ができるようになっています。疑似的にSeverityがAlarm, Warning, Informationの3つを出力するボタンを付属しています。
* FB_TcArgument を通じて、状況に応じた付加的な2つの引数データをアラームテキストに結合しています。本サンプルコードでは、乱数で発生したエラーコードを表示させています。
* Visualization向けに用意された [イベントテーブル](https://infosys.beckhoff.com/content/1033/tc3_plc_intro/3524166155.html?id=4373836669159094324) はUnicode非対応のため、Tableコントロールを用いてアラーム画面を実装。
* `InterfaceEventViewer` インターフェースを通じてビューを更新。Visualization以外にも汎用的なビューを実装して付け替えることが可能。

https://github.com/user-attachments/assets/2241fa2c-d231-45c4-8543-84f3b86c350f

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

#### アラーム初期化

まず、PLC スタート後1サイクルだけ実行されるプログラムで、初期化を行います。
アラーム制御オブジェクトは、`fb_observer.event_table` ( `FB_TcEventTable` ) 内に格納されています。
この中の、`init_instance()` メソッドを使って登録したイベントクラスのイベントと関連付けやオプションを次の通り引数で定義します。

|引数|型|説明|
|--|--|--|
|eventClass|GUID|イベントクラスのGUIDを指定します。`TC_EVENT_CLASSES`に続いて作成したイベントクラス名を指定します。|
|nEventId|UDINT|登録したイベントID|
|eSeverity|[TcEventSeverity](https://infosys.beckhoff.com/content/1033/tc3_eventlogger/5286329611.html?id=7153189202325781619)|重要度|
|bWithResetOperation|BOOL|新規アラームと確認済みアラームの区別が必要か否か|
|bWithResetOperation|BOOL|アラーム解除のリセット操作が必要か否か|

サンプルプログラムではPLCスタート後一度だけ、 `GVL.AlarmDatabase` で定義したものを順次展開するプログラムとしています。

``` pascal
PROGRAM MAIN
VAR
    // Alarm calculation function block
    init : BOOL;
    fb_observer : FB_EventObserver;
    event_table_view : FB_Tc3EventVisualizationView;
    i : UINT;
END_VAR

(*
    FB_Observer内の FB_Alarm インスタンス配列に対して、GVL.AlarmDatabseで定義したイベントクラスの情報で紐付ける。
*)
IF NOT init THEN
    FOR i := 1 TO AlarmEventParam.ALARM_MAX_COUNT DO
        // init_instance() 
        fb_observer.event_table.init_instance(
            eventClass := TC_EVENT_CLASSES.UserEventClass,
            nEventId := GVL.AlarmDatabase[i].nEventId,
            eSeverity := GVL.AlarmDatabase[i].eSeverity,
            bWithConfirmation := GVL.AlarmDatabase[i].bWithConfirmation,
            bWithResetOperation := GVL.AlarmDatabase[i].bWithResetOperation
        );
    END_FOR
    // イベント集計結果を表示する `InterfaceEventViewer` の実装オブジェクトをセットします。
    // ここでは Visualization のビューに特化した `FB_Tc3EventVisualizationView` のインスタンスを与えます。
    fb_observer.event_table.viewer := event_table_view; 
    init := TRUE;
END_IF
```

#### アラーム発報御部

次に個々のアラームの発報制御プログラムを記述します。

まず、アラームの制御オブジェクトへの参照を `get_alarm(<init_instance()を使って登録した順番>)` を使って取り出し、操作を行います。
もしくは、先ほどの初期化メソッド `event_table.init_instance()` を実行した際の戻り値には、アラーム制御オブジェクトのポインタが戻ります。`TC_Alarm`へのポインタ型の配列を用意して格納した上で、こちらからお使い頂く方法でも構いません。

取り出したアラームオブジェクト `alarm_instance` は、基本的に次のBOOL型の入力変数を制御することでアラーム発報、確認、解除状態を変更できます。

|変数|説明|備考|
|--|--|--|
|set_activate|TRUEにセットしている間アラーム発報状態として通知する|初期化時に `bWithResetOperation` がFALSEの場合、本変数をFALSEにすると自動解除される。|
|set_confirm|TRUEへの立ち上がりで確認済み状態へ遷移する|初期化時に `bWithConfirmation` がTRUEの場合のみ有効。|
|set_clear|TRUEへの立ち上がりでアラーム解除する。ただし、set_activateがTRUEの場合は解除できない。|初期化時に `bWithResetOperation` がTRUEの場合のみ有効。|

また、インターロックを設けるためアラームが発生している状態かどうかを調べるには、`alarm_instance.bActie` プロパティを参照してください。

ラダープログラムにおいては、 `set_activate` をコイルとして出力するだけでアラーム制御可能です。また、`bWithResetOperation` がTRUEでラッチされたアラームである場合は、別途 `set_clear` をTRUEにしてください。

#### エラーコード等を付加するには

アラームテキストに、エラーコード等を付加するには、Event Class に登録したEventのDisplay text の {0} {1} に展開する文字列定義を行います。
下記のプログラム例は、乱数を生成して疑似エラーコードをテキスト出力する例です。

アラーム制御オブジェクト `alarm_instance` には {0}, {1}, {2}, .... に対して順に値を登録するための `add_arguments` メソッドを用意しています。
次の型に応じたジェネリック型 `T_Arg` を引数に持ちます。こちら[`T_Arg help functions`](https://infosys.beckhoff.com/content/1033/tcplclib_tc2_utilities/35240075.html?id=8764599832541673086)をご参照の上、表示したいもののデータ型に応じたT_Arg型への変換を行ってください。

* E_ArgType.ARGTYPE_BYTE
* E_ArgType.ARGTYPE_WORD
* E_ArgType.ARGTYPE_DWORD
* E_ArgType.ARGTYPE_REAL
* E_ArgType.ARGTYPE_LREAL
* E_ArgType.ARGTYPE_SINT
* E_ArgType.ARGTYPE_INT
* E_ArgType.ARGTYPE_DINT
* E_ArgType.ARGTYPE_USINT
* E_ArgType.ARGTYPE_UINT
* E_ArgType.ARGTYPE_UDINT
* E_ArgType.ARGTYPE_STRING
* E_ArgType.ARGTYPE_BOOL
* E_ArgType.ARGTYPE_ULARGE
* E_ArgType.ARGTYPE_LARGE

``` pascal
VAR
    alarm_instance      : REFERENCE TO FB_Alarm;
    error_argument      : INT := 0;
    error_code_random   : DRAND;
    tmp_string          : T_MaxString;
END_VAR

// アラーム1 （Alarmレベル） 発報制御
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

// アラーム2（Warningレベル）発報制御
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


// アラーム3（Informationレベル） 発報制御
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

#### 一斉アラーム確認、一斉アラーム解除ボタン制御

アラーム確認ボタン(`confirm_button_input`)の受付と、アラームリセットボタン(`reset_button_input`)の受付のプログラムを定義します。
FB_TcEventTable には一斉にアラームを確認済みにする`confirm_all_alarm()`メソッドと、解除可能なアラームを解除する`try_clear_all_alarm()`メソッドが用意されています。これを使ってボタン操作により解除、確認処理を行います。

``` pascal
// アラームリセットボタン受付
IF reset_button_input THEN
    fb_observer.event_table.try_clear_all_alarm();
END_IF

// アラーム確認ボタン受付
IF confirm_button_input THEN
    fb_observer.event_table.confirm_all_alarm();
END_IF
```

#### Visualizationへの表示時刻のタイムゾーン設定

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

#### アラームテキストの言語設定反映

常時実行タスクが全て定義されている `FB_EventObserver` を実行します。また、Visualizationからのコンボボックス設定により表示させるアラームの言語コードを切り替えます。
直ちに表示が切り替わるのではなく、以後新規で発生するアラームテキストが、設定した言語に基づいて表示されます。

``` pascal
// オブザーバ（監視）オブジェクトの実行
CASE lang_select_options OF
    0: fb_observer.nLangId := 1033;
    1: fb_observer.nLangId := 1041;
END_CASE
fb_observer(event_class := TC_EVENT_CLASSES.UserEventClass);
```

#### アラーム操作ボタン、カウント表示

Visualizationに表示されるボタンの点滅、点灯制御などを定義します。Visualization上のパーツとの変数の関連は以下の通りです。

|変数|データ型|説明|
|---|---|---|
|status_unconfirm|ブール|「確認」ボタンの赤点灯・点滅表示|
|status_resetable|ブール|「リセット」ボタンの赤点灯表示|
|unconfirmed_count|値|未確認アラーム数|
|active_count|値|発生中アラーム数|

下記サンプルコードをご参考に、シグナルタワーの制御等へお役立てください。

``` pascal
// 表示用の変数制御（アクティブアラーム数、未確認アラーム数、ボタン点滅制御、リセット可能状態）
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

#### 履歴のクリア

「履歴消去」ボタン入力をMAINプログラムの `history_clear_button_input` に割り当てています。下記の通り5秒長押しすることで履歴消去を行うメソッドを呼び出します。

``` pascal
VAR
    history_clear_button_input	: BOOL;
    history_clear_delay_timer	: TON;
END_VAR

// 履歴の消去(5秒長押し)
history_clear_delay_timer(IN := history_clear_button_input, PT := T#5S);
IF history_clear_delay_timer.Q THEN
    fb_observer.event_table.viewer.history_clear();
END_IF
```

## Visualizationの所在

PLCプロジェクトの `VISUs` / `Visualization` が親画面となり、現在発生中アラームは`current`、アラーム履歴は`history`で個画面となっていて、タブ切り替えできる仕様です。

## その他注意

Visualizationに表示するアラーム一覧、履歴テーブルは、`FB_Tc3EventVisualizationView` の出力変数に割り当てられています。このテーブルデータはPERSISTENT属性が付加されていますので、正常にシャットダウンした場合に限り、永続化されています。

意図しないシャットダウン時にも直前の状態を保持するためには、UPSおよびPERSISTENT変数を保持するためのファンクションブロック制御が必要です。詳細は下記をご覧ください。

[https://beckhoff-jp.github.io/TwinCATHowTo/data_persistence/index.html](https://beckhoff-jp.github.io/TwinCATHowTo/data_persistence/index.html)

``` pascal
FUNCTION_BLOCK FB_Tc3EventVisualizationView IMPLEMENTS InterfaceEventViewer
VAR CONSTANT
    DISP_ROWS : UDINT := 80;
END_VAR
VAR_INPUT
    tzinfo : ST_TimeZoneInformation;
END_VAR
VAR_OUTPUT PERSISTENT
    stAlarmEvents	: ARRAY [1..DISP_ROWS] OF ST_EventView;
    stAlarmhistory 	: ARRAY [1..DISP_ROWS] OF ST_EventView;
END_VAR
VAR
    row : UDINT := 1;
END_VAR
```