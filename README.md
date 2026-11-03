# box_mission：画像距離推定＋直進接近版

ROS 2 Humble / Gazebo Classic。既存のbox_missionとfield_worldを変更したコード一式です。
Florence-2は既存のflorence2_workをそのまま呼び出します。GitHubへのpushはしていません。

## 動作

1. 初期goalへNav2で移動し、左90°で撮影・OCR、右180°。
2. a/b/c/dに対応した経路をFollowWaypointsで走行。
3. 経路終点をSpin指令角度0°とし、0/15/30/45/60/75°の6枚を撮影。
4. 最初に青箱を検出した写真を採用。その撮影方向へ戻る。
5. 新しい写真でbbox中心から旋回角を計算し、Spinと再撮影で中央へ合わせる。
6. bbox高さから箱までの距離を推定。両costmapのobstacle_layerをfalseにする。
7. 予定接近距離の半分まで、0.15 m/sで直進。オドメトリで距離を測る。
8. ゼロ指令を送り、オドメトリで停止を確認して撮影。未検出なら左10°・右10°も撮影。
9. 再び箱へ向きを合わせて距離を推定し、直進を再開。
10. 正の前進指令に対するCollision Monitorのゼロ出力が継続したら、前進指令を解除。
11. WAIT_CONFIRMで人の確認を待つ。自動では再発進しない。
12. 確認後にSpinの累積指令角度を相殺し、両レイヤーをtrueへ戻してnext_fileを呼ぶ。

半分に達する前にMonitorが停止させた場合は、途中撮影を省略してWAIT_CONFIRMへ移ります。
Monitorのゼロ出力は「青箱へ到達した証拠」ではありません。他の障害物でも止まるため、人が到達を確認します。
途中撮影は1回です。再推定距離の半分で繰り返し撮影する処理ではありません。
初期撮影・分岐経路にはNav2を使用し、箱へ接近する区間にはNavigateToPoseを使用しません。
Bの終了監視そのものは従来どおり含みません。B終了後にmission.pyを起動します。

## 同梱内容と変更箇所

| パス | 内容 |
|---|---|
| box_mission/mission.py | 全体フロー、撮影切替、Spin、レイヤー・人の確認・次経路 |
| box_mission/visual_approach.py | ROS画像、Florence呼出、再照準、オドメトリ直進、停止検知 |
| box_mission/approach_logic.py | bbox選択・角度・距離・接近距離の計算 |
| box_mission/box_hooks.py | Florence結果をミッションへ渡す接続 |
| box_mission/config.yaml | Gazebo用の既定設定 |
| box_mission/routes_field/ | フィールド寸法に合わせた撮影地点と4経路の初期案 |
| box_mission/test_mission.py | ROSなしの25件の単体・模擬制御試験 |
| field_world/field_world/command_gate.py | Nav2/直進指令の排他切替と入力失効処理 |
| field_world/launch/mission_nav.launch.py | Nav2、ゲート、Collision Monitor、Lifecycle起動 |
| field_world/launch/sim_nav.launch.py | 上記launchを使用するよう変更 |
| field_world/config/collision_monitor.yaml | Humble用の停止領域・/low_scan入力 |
| field_world/config/nav2_params_odom_only.yaml | Spin明示、標準Nav2 BT使用 |
| field_world/setup.py / package.xml | 実行エントリと依存関係の追加 |

field_worldのモデル・テクスチャ・ツールも同梱しています。
florence2_workは同梱していません。既存の仮想環境・モデルを使用します。
既存のbox_mission/routes/は元の仮座標のまま保存しています。既定ではroutes_field/を使います。
各パッケージのREADME_before.mdは変更前の説明です。

## 導入

既存のorne-boxおよびGazebo依存関係はセットアップ済みであることを前提とします。
既存ファイルをバックアップし、このZIPのfield_worldをROSワークスペースのsrcへ、
box_missionをホームディレクトリなどへ配置してください。field_worldを同名で二重に配置しないでください。

```bash
source /opt/ros/humble/setup.bash
sudo apt install python3-opencv python3-yaml \
  ros-humble-cv-bridge ros-humble-nav2-msgs ros-humble-lifecycle-msgs \
  ros-humble-nav2-collision-monitor

cd ~/ros2_ws
colcon build --packages-select field_world --symlink-install
source install/setup.bash
```

florence2_workは従来の環境を利用します。config.yamlの以下の4パスを実際の環境へ合わせます。

```yaml
ocr:
  python: ~/florence2_work/.venv/bin/python
  script: ~/florence2_work/florence2_ocr.py
detector:
  python: ~/florence2_work/.venv/bin/python
  script: ~/florence2_work/florence2_phrase.py
```

モデルをローカルに保存している場合はocr.model / detector.modelに絶対パスを指定してください。
既定のmodel空文字は各スクリプトの既定モデルを使用します。
ROS側はsystem Pythonで起動します。ROS側でFlorenceのvenvをactivateする必要はありません。

## 起動

ターミナル1：シミュレータ、ロボット、Nav2、停止系。

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 launch field_world sim_nav.launch.py seed:=42 green_label:=c
```

この版ではvelocity_smootherを起動せず、次の経路を使います。

| 指令元 | 経路 |
|---|---|
| Nav2 controller / behavior | /cmd_vel_nav_stack → box_command_gate → /cmd_vel_raw |
| 箱へ直進するmission | /box_mission/cmd_vel → box_command_gate → /cmd_vel_raw |
| 最終出力 | /cmd_vel_raw → Collision Monitor → /cmd_vel → ロボット |

/cmd_velのpublisherはCollision Monitorだけにします。
以前のNav2 bringup、velocity_smoother、別のCollision Monitor、teleopを重ねて起動しないでください。
起動後は`ros2 topic info /cmd_vel -v`、`ros2 lifecycle get /collision_monitor`で接続とactive状態を確認できます。
missionのpreflightでも最終速度publisherがCollision Monitorの1つだけか確認します。

ターミナル2：通常のwaypoint_sender_nodeを起動してBの終了位置で待機させます。
`/waypoint_sender_node/next_file`サービスが必要です。field_world単体ではこのサービスを作りません。
元のwaypoint_toolsの起動方法と経路構成を使用してください。ミッション中は他のゴールを送らないでください。
末尾のnext_fileは実際の次経路を送る処理です。成功を返すだけのダミーサービスは同梱していません。

ターミナル3：B終了後にミッションを開始。

```bash
source /opt/ros/humble/setup.bash
source ~/ros2_ws/install/setup.bash
cd ~/box_mission
python3 mission.py --config config.yaml
```

到達して停止したことを確認後、別ターミナルから実行します。

```bash
ros2 service call /box_mission/confirm_arrival std_srvs/srv/Trigger '{}'
```

走行中・探索中の確認入力は拒否します。受理の応答は全処理の完了ではありません。
STATE: DONEで復帰・next_file要求の成功を確認します。後続経路の完走までは待ちません。

```bash
ros2 topic echo /box_mission/state
ros2 topic echo /box_command_gate/status
```

## 距離と旋回の定義

bbox中心をu、主点をcx、焦点距離をfxとすると、Spin角度は
`atan2(cx - u, fx)`です。画像左がROSの正の旋回です。
角度許容値は2°、1回の補正は最大20°、補正回数は5回です。補正のSpinも累積指令角度へ加えます。

カメラから箱までの光軸距離Dは`fy * box_height_m / bbox_height_px`で概算します。
箱全体が写っている必要があります。画像端に接するbboxや設定距離範囲外の結果は拒否します。

予定接近距離Lは以下です。

`L = D + camera_forward_m - robot_front_m - stand_off_m`

既定は、箱高さ0.31m、カメラ前方位置0.4m、ロボット前端0.6m、手前余裕0.3mです。
例えばD=4mならL=3.5m、その半分の1.75mで撮影します。
実際の停止位置はCollision Monitorの領域と点群で決まります。
後半は再推定L+final_extra_m（既定0.8m）を最大走行距離とし、Monitor停止が無いまま上限へ達したら異常終了します。
総接近距離の上限も適用します。距離推定の値だけで到達認定しません。
オドメトリ距離は直進開始時の向きへ投影した変位です。曲線経路の長さではありません。

## カメラ・箱・候補選択の前提

- 現在の距離式は水平、前向き、車体中心線上、非鏡像のカメラ用です。
- Gazeboの既定camera_x=0.40、camera_y=0、camera_pitch=0に対応します。
- カメラ位置を変更したらcamera_forward_mなども変更してください。URDFとの自動照合はしません。
- camera_pitchを変更する場合は、pitchを含む幾何への拡張が必要です。この版では未対応です。
- ROS画像ではcamera_infoのKを使用します。画像サイズとframe_idを確認し、歪み係数が非ゼロなら拒否します。
- USBではcamera.sourceをusb、use_sim_timeをfalseとし、camera.intrinsicsに校正済みの値を設定します。
  同梱のintrinsicsはGazebo相当の参考値です。実機への流用はできません。
  USB入力はこの版では歪み補正しないため、補正済み相当の画像と内部パラメータが必要です。
- 実箱の高さは実測値へ変更します。bboxの高さに上面や背景が含まれると距離誤差が出ます。
- 青色フィルタはHSV既定H=90..135、S>=60、V>=30、bbox内青色割合>=0.08です。照明に応じて調整します。
- 6枚からは最初に検出した写真、1枚の複数候補からは最大bboxを選びます。
  物体ID追跡や青箱ラベルの照合は実装していません。目的の青箱が1つ写る条件を基本にしてください。
  隣接エリアの青箱も写る場合は対象を取り違える可能性があり、別途候補の識別が必要です。

## 停止と旋回

FrontStopは車体前端より0.3m前方までの前進停止領域、FootprintStopは車体周囲の保護領域です。
Humble版の仕様に合わせ、pointsはdouble配列、点数閾値名はmax_pointsです。
前方の箱がFrontStopに入ったままだとSpinも止まるため、missionがSpinする間だけFrontStop.enabledをfalseにします。
FootprintStopとMonitor自体は無効にしません。Spin終了・失敗時にはFrontStopをtrueへ戻します。
保護領域は同梱のfootprintに合わせた初期値です。実機形状・旋回時の張り出し・センサ死角は別途確認してください。
/low_scanは前方180°なので、後方の障害物まで監視できる構成ではありません。

HumbleのMonitorに専用状態topicが無い構成でも使えるよう、直進中の/cmd_velのゼロ出力継続で停止を推定します。
これは停止原因の確定ではありません。センサ欠落やゲート異常は別途検知して異常終了します。
初期ゼロ出力を誤検知しないため0.6秒の猶予、その後0.25秒の継続判定を行います。

ゲートは入力が0.4秒途絶したらゼロを送ります。mission所有中はNav2へ自動で戻しません。
low_scanの受信・stampが古い場合もゼロにします。直進側はodom・Monitor出力・ゲート状態を監視します。
停止時はオドメトリ速度が並進0.02m/s未満・角速度0.03rad/s未満で継続することも確認します。
Gazeboを長くpauseするとタイムアウトで終了します。

異常終了時は直進を停止し、自分のNav2 Actionをキャンセル、レイヤーを変更前の値へ戻します。
逆回転・next_fileは実行しません。接近で取得したゲート所有権は、異常時には自動解除しません。
再実行前に停止原因とNav2ゴールが残っていないことを確認して、次を実行します。

```bash
ros2 param set /collision_monitor FrontStop.enabled true
ros2 service call /box_command_gate/mission_control std_srvs/srv/SetBool '{data: false}'
```

SIGKILLなどPythonの終了処理が走らない停止では、パラメータ復帰処理も実行されません。再起動時は上記を確認します。
ゲート・Monitor・ロボットの指令タイムアウト設定も含めて実環境で確認してください。

## 検証状況

```bash
cd ~/box_mission
python3 test_mission.py
```

25件の単体・模擬試験が成功しています。
角度の符号、bbox距離、半分地点、途中探索、逆回転、早期停止、人の確認の受付制限、
前進停止、odom・出力途絶、センサ欠落、移動不能、ゲートの入力失効などを確認しました。
Python構文とYAMLの読み込みも確認しています。

作成環境にはROS 2/Gazebo/OpenCV/Florence推論環境がなく、ROS通信・Gazebo起動・実推論・実走行は未検証です。
routes_fieldはフィールド寸法から作った初期案です。経路の走行成功、カメラ撮影品質、ランダムseed全体の認識成功を保証するものではありません。
最初に固定seed・固定文字で画像と停止位置を確認し、その後にa/b/c/dと複数seedで評価してください。
認識の検証ではanswer.yamlを制御入力に使いません。

## 元のソース

- box_mission: https://github.com/Shuma-s58/box_mission （041bc296f471b9d3f12cb04bc93e3fd8ca250903）
- field_world: https://github.com/Shuma-s58/field_world （92792a963522749c81dd95fe59d878c9e910e1e5）
- florence2_work: https://github.com/Shuma-s58/florence2_work （ef7d4f317f8508bce5404b7b0f667c984c5b7471）
- HumbleのMonitor設定・速度経路の参照: https://github.com/ros-navigation/navigation2/tree/humble
  （確認コミット3c3db59d6969d8ecee8e68468693d006397f4a0c）
