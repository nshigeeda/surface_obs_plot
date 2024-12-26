# surface_obs_plot
実況気象通報式（SYNOP）・地域気象観測（アメダス）プロット<br>
2024_Dec_20作成, 2024_Dec_26更新, N.Shigeeda

# はじめに
このPythonコードは、実況気象通報式（SYNOP）と地域気象観測（アメダス）の実況観測データを取得し、これらのデータに基づく天気記号を地図上にプロットし、地上局地天気図（プロット図）を作成します。このプロット図を用いて、地上の等圧線、温度分布、シアラインの主観解析を行うことが可能です。<br>
　地上局地プロット図は東京学芸大学気象情報頁 専門天気図からDL可能ですが、このプロット図は北緯30度から44度付近をカバーする少し大きめの図です。例えば関東甲信地方に着目した局地解析を行いたい場合など、もう少し縮尺の小さいプロット図を使いたいことがあります。このような背景から、独自に指定する緯度経度範囲でプロット図を作成するコードを実装しました。<br>
　局地解析は、地形や降水などの実況も重ね合わせて行うことが有効です。地形図や雨雲合成レーダーなど他の情報をプロット図に重ね合わせることで、局地解析の精度を上げることが可能です。本コードはプロット図を背景透明な画像として保存するので、別途入手した他の画像（図）を重ね合わせて見ることができます。<br>
　地図の縮尺を小さくするとSYNOPの通報地点が相対的に減ります。そこで補完的にアメダスの実況観測データに基づく天気記号も同じ地図上にプロットします。これによりある程度精度のある局地解析を行えるようにしました。<br>
　SYNOPにおける風速は国によって運用が異なり、単位に関する情報も風速と一緒に通報されます。日本ではノットを使用しますが、中国・米国はm/sです。本コードでは、風速はノットをm/sに換算してプロットします。なおアメダスの風速は元からm/sで取得できるので単位の変換はしていません。風向は、SYNOPは36方位、アメダスは16方位です。それぞれ角度にした値をプロットします。SYNOPは、短矢羽：5m/s、長矢羽：10m/s、旗矢羽：50m/sです。アメダスは、短矢羽：1m/s、長矢羽：2m/s、旗矢羽：10m/sです。なおアメダスについては、wind_mag パラメータで強調度を変更可能です。<br>
　METARやSHIPも、SYNOPと同様にしてプロットすることは可能と思われますが現時点で未実装です。<br>
 
## 実況気象通報式について
1. データソースあれこれ<br>
    実況気象通報式は次のサイトから入手可能です。それぞれ良し悪しがあるので、目的に応じて扱いやすい情報ソースを選択する必要があります。主観的になりますが、今回のプロット図作成を念頭に置いて利点と欠点をあげます。
    - NCEI NOAA（https://www.ncei.noaa.gov/data/global-hourly/）<br>
    **［利点］** 過去の実況気象データを入手できる。観測が始まっていない等でデータのある地点は限られますが、一番古いデータは1901年からあります。観測データはcsv形式になっており、デコードされた値が格納されています。風速はm/sです。仕様書が整備されています（https://www.ncei.noaa.gov/data/global-hourly/doc/）<br>
    **［欠点］** 地点毎、1年分のデータが一つのファイルに収まっています。プロット図のように複数地点をプロットするには複数のファイルをDLし、対象時刻の観測データを選択的に抽出する必要があります。ファイル名は地点をユニークに識別するコードが使われていますが、国際地点番号の前に米国空軍の定義したコードがつけられているので、ファイル名を選択するのも工夫が要ります。<br>
    - Unidata（https://thredds.ucar.edu/thredds/catalog/catalog.html）<br>
    **［利点］** 過去1か月前までの SYNOP, SHIP, SYNOP MOBIL, METARが入手可能。但しMETARは別ディレクトリ。年月日時を単位とするtextファイルに複数地点のデータが格納されているので、プロット図を作成する際は一つのtextファイルをDLすれば済みます。データの提供システムはThredds data server(TDS)なので、siphonライブラリを使ってAPI経由でデータを取得可能。<br>
    **［欠点］** 最新データの反映が遅い時がある。データの反映が遅いときは、昨日今日の観測に基づくプロット図を作成することはできません。データはWMOの仕様に基づいてSYNOPを包む形でエンコードされているらしく、SYNOPを扱うためには、まずSYNOPデータを取り出す必要があります。※METARはデータの抽出とデコードまでを一貫して行うAPIがMETPYに用意されている。<br>
    - 東京学芸大学気象情報頁 専門天気図（http://tenki.u-gakugei.ac.jp/advanced.html）<br>
    **［利点］** 最新データの反映が一番早く、昨日今日の観測に基づくプロット図を作成できます。データは、00/03/06/09/12/15/1/8/21UTC毎のtextファイルになっており、ひとつのtextファイルに複数地点のSYNOPが格納されているので、対象時刻のプロット図を生成する時はひとつのファイルをDLすれば済みます。<br>
    **［利点］** データは過去1週間前までしか遡れないため、それ以前の災害事例などを取り上げて解析したい場合は、別のデータソースを頼るしかありません。<br>

2. 実況気象通報式のデコードについて<br>
    SYNOP報のデコードには pymetdecoder というライブラリが利用できます（https://pypi.org/project/pymetdecoder/）。<br>
    - このライブラリはそこそこの頻度で更新されているようで、2024.12.25時点で v0.1.6が利用可能です。SYNOP (FM-12), SHIP (FM-13), SYNOP MOBIL (FM-14)に対応しています。pymetdecoderは、風向を角度にデコードしてくれます。なおアメダスについては本コード内で16方位を角度に変換しています。<br>
    - ところでMETARについては、Metpy の parse_metar_file というメソッドを用いると簡単にデコードしてくれます。(https://unidata.github.io/MetPy/latest/api/generated/metpy.io.parse_metar_file.html) 本コードではMETARのプロットはしていないため使用していません。このようなメソッドがMETARにはあって、なぜSYNOPにはないのかはわかりません。<br>

3. 緯度経度情報について<br>
    - SYNOPには緯度経度情報が含まれないません。SYNOPについては、定置地上観測所の識別コードをキーにして、国際地点番号表から緯度経度情報を求めています。<br>

