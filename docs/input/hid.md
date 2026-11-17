# HID: あらゆるキーボード、マウス、ゲームパッドのための1つのドライバ {#hid-one-driver-for-every-keyboard-mouse-and-gamepad}

> デバイスは自分自身のデータシートを持ち歩いている。カーネルが接続時にパースするレポートディスクリプタである。そのため、1つの汎用ドライバでボタンの付いたほぼあらゆるものを扱える

## 問題: キーボードはネットワークカードではない {#the-problem-a-keyboard-is-not-a-network-card}

ネットワークカードのドライバは自分のチップを知っていなければならない。どのレジスタがDMAリングを保持するか、リンクパートナーのオートネゴシエーション状態をどう読むか、割り込み要因のビットマップが何を意味するか、といったことである。その知識は1つのチップファミリーに固有のもので、ファミリーごとに一度ずつ手で書かなければならない。もし入力デバイスも同じ仕組みだったら、Linuxはこれまでに作られたあらゆるマウス、キーボード、ゲームパッド、ペンタブレット、音量調整ノブのそれぞれに専用ドライバを必要とすることになる。レポートのどのバイトが「左ボタン」を意味するかだけが異なる、ほとんど同一の小さなドライバが何千個も必要になるのである。

1996年に承認されたUSBのHuman Interface Device（HID）クラスは、まさにそれを避けるために設計された。ホストがデバイスのデータ形式をあらかじめ知っている必要はなく、*デバイス*のほうが自身のレポートレイアウトのコンパクトなバイトコード記述、つまり**レポートディスクリプタ**を持ち歩き、列挙の際にそれをホストに渡す。HIDのホストスタックはそのディスクリプタを一度読み、デバイスがこれから送受信するすべてのレポートの正確なビットレイアウトを把握する。それ以降は、ドライバの作者が一度も見たことのないデバイス、聞いたこともないベンダーが作ったデバイスからのトラフィックでもデコードできる。このクラスは後に、ほぼそのままの形でBluetooth（HID Profile、HIDP）やI2C（ノートPCやタブレットでよく見られる組み込みのタッチパッドやタッチスクリーン向け）に移植された。自己記述的なレポートという同じ考え方が、USBと同様にそれらのリンク上でも有用だからである。

`drivers/hid/` を根とするLinuxのHIDサブシステムは、この同じ考え方を中心に構築されている。レポートディスクリプタのパース方法と、生のレポートバイトを型付きのフィールド値に変換する方法を知っている、トランスポートに依存しない1つのコア（`drivers/hid/hid-core.c`）、それらの型付きの値をevdevイベントに変換する1つの汎用入力マッピング層（`drivers/hid/hid-input.c`）、そしてバイトのやりとりの方法だけを知っていればよい、トランスポートごとの薄いシム（USB-HID、BT-HIDP、I2C-HID）である。普通のマウスやキーボードには、ベンダー固有のドライバはまったく必要ない。`hid-generic` がコアのパースできるあらゆるデバイスにマッチし、汎用テーブルにすべての仕事をさせる。`drivers/hid/` 以下にある150を少し超える小さなquirkドライバのいずれかが必要になるのは、レポートディスクリプタが壊れているか通常と異なるデバイスや、汎用マッピングテーブルがまだ知らないものを公開したいデバイスだけである。

## レポートディスクリプタ: デバイス自身のデータシート {#report-descriptors-the-devices-own-datasheet}

レポートディスクリプタは、スタックベースでTLVに似たバイトコードで書かれた短いプログラムである。各**アイテム**は1〜5バイトで、アイテムの*型*と*タグ*をエンコードした1バイトのプレフィックスに、0、1、2、または4バイトの値が続く。アイテムには3つの型があり、パーサ（後述）はその型でディスパッチする。

- **Mainアイテム** — `Input`、`Output`、`Feature`、`Collection`、`End Collection`。実際にフィールドを出力したり、グループを開閉したりするものである。
- **Globalアイテム** — `Usage Page`、`Logical Minimum`/`Maximum`、`Report Size`、`Report Count`、`Report ID`、`Push`/`Pop`。変更されるまでアイテムをまたいで保持される状態であり、次の `Input`/`Output`/`Feature` アイテムが解釈される文脈を形作る。
- **Localアイテム** — `Usage`、`Usage Minimum`/`Maximum`、`Designator Index`、`String Index`。*次の*Mainアイテムにだけ適用され、その後クリアされる状態である。

`include/linux/hid.h` には、パーサがディスパッチに使う正確なタグ値が書かれている。

```c
// include/linux/hid.h
#define HID_MAIN_ITEM_TAG_INPUT			8
#define HID_MAIN_ITEM_TAG_OUTPUT		9
#define HID_MAIN_ITEM_TAG_FEATURE		11
#define HID_MAIN_ITEM_TAG_BEGIN_COLLECTION	10
#define HID_MAIN_ITEM_TAG_END_COLLECTION	12

#define HID_GLOBAL_ITEM_TAG_USAGE_PAGE		0
#define HID_GLOBAL_ITEM_TAG_LOGICAL_MINIMUM	1
#define HID_GLOBAL_ITEM_TAG_LOGICAL_MAXIMUM	2
#define HID_GLOBAL_ITEM_TAG_REPORT_SIZE		7
#define HID_GLOBAL_ITEM_TAG_REPORT_ID		8
#define HID_GLOBAL_ITEM_TAG_REPORT_COUNT	9
#define HID_GLOBAL_ITEM_TAG_PUSH		10
#define HID_GLOBAL_ITEM_TAG_POP			11

#define HID_LOCAL_ITEM_TAG_USAGE		0
#define HID_LOCAL_ITEM_TAG_USAGE_MINIMUM	1
#define HID_LOCAL_ITEM_TAG_USAGE_MAXIMUM	2
```

**Usage Page** と **Usage** は、フィールドに単なるビット位置ではなく意味を与えるものである。Usage Pageは名前空間であり、`Generic Desktop`（物理的なデバイスクラスとしてのマウス、キーボード、ジョイスティック、およびX/Y/Wheelのような軸のusage）、`Keyboard/Keypad`、`Button`、`Consumer`（メディアキー）、`LED` など数十種類があり、すべてUSB-IFが公開している *HID Usage Tables* に列挙されている。Usageはそのページ内の特定の意味であり、Usage Page `Keyboard/Keypad`（0x07）のUsage `0x04` は文字どおり「Keyboard a and A」である。組み合わせると、32ビットのusage値（`include/linux/hid.h` の `HID_USAGE_PAGE`/`HID_USAGE` マスクに従い、上位16ビットがページ、下位16ビットがusage）は、「このフィールドはこれを意味する」ことを表す、グローバルに一意な名前となる。

**Collection** は関連するアイテムをグループ化する。`Application`（「この全体がマウスである」というトップレベルのグループ）、`Physical`、`Logical`、`Report` などがあり、ネストできる。そのため、ゲームパッドのディスクリプタは、アナログスティック用のPhysicalコレクションと、十字キー用の別のPhysicalコレクションを含む、トップレベルのApplicationコレクションを1つ持つ、といった形になりうる。

概念的には（生のバイトを手でデコードしたものではなく、構造的な形として）、最小限の3ボタンマウスのディスクリプタは次のようになる。

```
Usage Page (Generic Desktop)
Usage (Mouse)
Collection (Application)
    Usage (Pointer)
    Collection (Physical)
        Usage Page (Button)
        Usage Minimum (1)
        Usage Maximum (3)
        Logical Minimum (0)
        Logical Maximum (1)
        Report Count (3)          # 1ビットのボタンフィールドが3つ
        Report Size (1)
        Input (Data, Variable, Absolute)   # 3つのボタンのビット

        Report Count (1)          # バイトを埋めるための5ビットのパディング
        Report Size (5)
        Input (Constant)                    # パディング、無視される

        Usage Page (Generic Desktop)
        Usage (X)
        Usage (Y)
        Logical Minimum (-127)
        Logical Maximum (127)
        Report Size (8)
        Report Count (2)          # 1バイトのフィールドが2つ: X、Y
        Input (Data, Variable, Relative)   # X/Yの移動量のバイト
    End Collection
End Collection
```

`Input`/`Output`/`Feature` アイテムはそれぞれ1つの**フィールド**を確定させる。その時点で有効な `Usage`、`Logical Minimum`/`Maximum`、`Report Size`、`Report Count` がそのフィールドの形となり、アイテム自身のフラグ（`Data`/`Constant`、`Variable`/`Array`、`Absolute`/`Relative`）がビットの解釈方法を示す。このマウスの例は3バイトの入力レポートを生成する。バイト0の下位3ビットがボタンの状態（その上に5ビットのパディング）、バイト1と2が符号付きの相対的なXとYの差分である。この形はどのカーネルドライバにもハードコードされていない。以下で説明するディスクリプタのパースが再構築するのは、まさにこの形である。

## パース: `hid_open_report()` と `struct hid_parser` {#parsing-hidopenreport-and-struct-hid_parser}

すべてのHIDドライバの `probe()` は、何よりも先に `hid_parse()`（`hid_open_report()` の薄いラッパー）を呼ぶ。

```c
// include/linux/hid.h
/**
 * hid_parse - parse HW reports
 *
 * @hdev: hid device
 *
 * Call this from probe after you set up the device (if needed). Your
 * report_fixup will be called (if non-NULL) after reading raw report from
 * device before passing it to hid layer for real parsing.
 */
static inline int __must_check hid_parse(struct hid_device *hdev)
{
	return hid_open_report(hdev);
}
```

`hid_open_report()`（`drivers/hid/hid-core.c`）は、トランスポートドライバがデバイスから読み取った生のディスクリプタのバイトを受け取り、ドライバの `.report_fixup` コールバックが登録されていればそれをコピーに対して実行し（これにより、本物のパーサが目にする前にquirkドライバが壊れたディスクリプタにパッチを当てられる。後述の実例を参照）、それから（パッチが当たっているかもしれない）バイトを `hid_parse_collections()` に渡す。

```c
// drivers/hid/hid-core.c
int hid_open_report(struct hid_device *device)
{
	unsigned int size;
	const u8 *start;
	int error;

	if (WARN_ON(device->status & HID_STAT_PARSED))
		return -EBUSY;

	start = device->bpf_rdesc;
	if (WARN_ON(!start))
		return -ENODEV;
	size = device->bpf_rsize;

	if (device->driver->report_fixup) {
		/*
		 * device->driver->report_fixup() needs to work
		 * on a copy of our report descriptor so it can
		 * change it.
		 */
		u8 *buf __free(kfree) = kmemdup(start, size, GFP_KERNEL);

		if (!buf)
			return -ENOMEM;

		start = device->driver->report_fixup(device, buf, &size);

		/*
		 * The second kmemdup is required in case report_fixup() returns
		 * a static read-only memory, but we have no idea if that memory
		 * needs to be cleaned up or not at the end.
		 */
		start = kmemdup(start, size, GFP_KERNEL);
		if (!start)
			return -ENOMEM;
	}

	device->rdesc = start;
	device->rsize = size;

	error = hid_parse_collections(device);
	...
	return 0;
}
```

`hid_parse_collections()` はバイトストリームをアイテムごとに走査し、アイテムの型をキーとする小さなルックアップテーブルを通じて各アイテムをディスパッチする。

```c
// drivers/hid/hid-core.c
static typeof(hid_parser_main) (* const dispatch_type[]) = {
	hid_parser_main,
	hid_parser_global,
	hid_parser_local,
	hid_parser_reserved
};
...
while ((next = fetch_item(start, end, &item)) != NULL) {
	start = next;
	...
	if (dispatch_type[item.type](parser, &item)) {
		hid_err(device, "item %u %u %u %u parsing failed\n", ...);
		goto out;
	}
}
```

`hid_parser_global()` は、`struct hid_parser` の `global` フィールド（および `Push`/`Pop` 用の `global_stack`）にある永続的な状態（現在のUsage Page、Report Size、Report Countなど）を更新する。`hid_parser_local()` は*次の*Mainアイテムに適用されるUsageを蓄積する。`hid_parser_main()` は、`Input`/`Output`/`Feature`/`Collection` アイテムが蓄積された状態を実際に消費し、デバイスの `struct hid_report` 内に `struct hid_field` を割り当てる（または `struct hid_collection` を開閉する）場所である。`hid_open_report()` が戻る時点で、ディスクリプタ全体がサブシステムの残りの部分が操作するオブジェクトグラフに変換されており、実際のレポートを読むホットパスでバイトレベルのパースがさらに行われることはない。

## `struct hid_device` とパース済みのオブジェクトグラフ {#struct-hid_device-and-the-parsed-object-graph}

`struct hid_device`（`include/linux/hid.h`）はパース済みのデバイスである。ディスクリプタのバイト、パーサが構築したレポートテーブル、トランスポートの配管、アタッチされたドライバが、すべて1つのオブジェクトに収まっている。

```c
// include/linux/hid.h
struct hid_device {
	const __u8 *dev_rdesc;						/* device report descriptor */
	const __u8 *bpf_rdesc;						/* bpf modified report descriptor, if any */
	const __u8 *rdesc;						/* currently used report descriptor */
	unsigned int dev_rsize;
	unsigned int bpf_rsize;
	unsigned int rsize;
	unsigned int collection_size;					/* Number of allocated hid_collections */
	struct hid_collection *collection;				/* List of HID collections */
	unsigned int maxcollection;						/* Number of parsed collections */
	unsigned int maxapplication;					/* Number of applications */
	__u16 bus;							/* BUS ID */
	__u16 group;							/* Report group */
	__u32 vendor;							/* Vendor ID */
	__u32 product;							/* Product ID */
	__u32 version;							/* HID version */
	enum hid_type type;						/* device type (mouse, kbd, ...) */
	unsigned country;						/* HID country */
	struct hid_report_enum report_enum[HID_REPORT_TYPES];
	struct work_struct led_work;					/* delayed LED worker */

	struct semaphore driver_input_lock;				/* protects the current driver */
	struct device dev;						/* device */
	struct hid_driver *driver;
	void *devres_group_id;						/* ID of probe devres group	*/

	const struct hid_ll_driver *ll_driver;
	struct mutex ll_open_lock;
	unsigned int ll_open_count;
	...
	unsigned long status;						/* see STAT flags above */
	unsigned claimed;						/* Claimed by hidinput, hiddev? */
	unsigned quirks;						/* Various quirks the device can pull on us */
	...
	struct list_head inputs;					/* The list of inputs */
	void *hiddev;							/* The hiddev structure */
	void *hidraw;

	char name[128];							/* Device name */
	char phys[64];							/* Device physical location */
	char uniq[64];							/* Device unique identifier (serial #) */
	u64 firmware_version;						/* Firmware version */

	void *driver_data;
	...
};
```

`report_enum[HID_REPORT_TYPES]` がパーサの本当の出力である。レポート型（`HID_INPUT_REPORT`、`HID_OUTPUT_REPORT`、`HID_FEATURE_REPORT`、`include/uapi/linux/hid.h`）ごとに1つの `struct hid_report_enum` があり、それぞれが `report_id_hash[]` を保持している。これにより、番号付きのレポート（1つのエンドポイント上で複数のレポート形状を多重化するデバイス向けに、各パケットの先頭にReport IDのバイトを付けるもの）を、生のパケットが届いたときにO(1)で検索できる。

`struct hid_report` は1つのパース済みレポートであり、`hid_open_report()` が、Report IDを共有する `Input`/`Output`/`Feature` アイテムの1つの区間から構築する単位である。

```c
// include/linux/hid.h
struct hid_report {
	struct list_head list;
	struct list_head hidinput_list;
	struct list_head field_entry_list;		/* ordered list of input fields */
	unsigned int id;				/* id of this report */
	enum hid_report_type type;			/* report type */
	unsigned int application;			/* application usage for this report */
	struct hid_field *field[HID_MAX_FIELDS];	/* fields of the report */
	struct hid_field_entry *field_entries;		/* allocated memory of input field_entry */
	unsigned maxfield;				/* maximum valid field index */
	unsigned size;					/* size of the report (bits) */
	struct hid_device *device;			/* associated device */
	...
};
```

そして `struct hid_field` は、ちょうど1つの `Input`/`Output`/`Feature` アイテム分の形、つまりレポート内のどこにあるか、どれだけの大きさか、何を意味するかを表す。

```c
// include/linux/hid.h
struct hid_field {
	unsigned  physical;		/* physical usage for this field */
	unsigned  logical;		/* logical usage for this field */
	unsigned  application;		/* application usage for this field */
	struct hid_usage *usage;	/* usage table for this function */
	unsigned  maxusage;		/* maximum usage index */
	unsigned  flags;		/* main-item flags (i.e. volatile,array,constant) */
	unsigned  report_offset;	/* bit offset in the report */
	unsigned  report_size;		/* size of this field in the report */
	unsigned  report_count;		/* number of this field in the report */
	unsigned  report_type;		/* (input,output,feature) */
	__s32    *value;		/* last known value(s) */
	__s32    *new_value;		/* newly read value(s) */
	...
	__s32     logical_minimum;
	__s32     logical_maximum;
	__s32     physical_minimum;
	__s32     physical_maximum;
	...
	struct hid_report *report;	/* associated report */
	unsigned index;			/* index into report->field[] */
	/* hidinput data */
	struct hid_input *hidinput;	/* associated input structure */
	...
};
```

そして `struct hid_usage` は、フィールドに付けられた1つのUsageである（1つのフィールドが複数持つこともある。配列フィールドや、Usage Minimum/Maximumの範囲から作られたフィールドは、複数のusageをカバーする）。

```c
// include/linux/hid.h
struct hid_usage {
	unsigned  hid;			/* hid usage code */
	unsigned  collection_index;	/* index into collection array */
	unsigned  usage_index;		/* index into usage array */
	__s8	  resolution_multiplier;/* Effective Resolution Multiplier
					   (HUT v1.12, 4.3.1), default: 1 */
	/* hidinput data */
	__s8	  wheel_factor;		/* 120/resolution_multiplier */
	__u16     code;			/* input driver code */
	__u8      type;			/* input driver type */
	__s16	  hat_min;		/* hat switch fun */
	__s16	  hat_max;		/* ditto */
	__s16	  hat_dir;		/* ditto */
	__s16	  wheel_accumulated;	/* hi-res wheel */
};
```

`usage->hid` はディスクリプタから直接得られる生のHID usageコード（上で説明したページ + usage）である。`usage->type`/`usage->code` は*後で*、evdevへのマッピングの際に埋められる。最初は意味を持たず、`hid-input.c` がこのusageがユーザー空間にとって何を意味するかを決めた時点で、`EV_*` 型と `KEY_*`/`ABS_*`/... コードが設定される（次のセクション）。`struct hid_collection` は4つの中で最も単純で、ネストを再構築するのに十分な情報だけを持つ。

```c
// include/linux/hid.h
struct hid_collection {
	int parent_idx; /* device->collection */
	unsigned type;
	unsigned usage;
	unsigned level;
};
```

## ドライバモデル: `struct hid_driver` {#the-driver-model-struct-hid_driver}

HIDドライバは、何でも受け止める `hid-generic` であれ、ベンダー固有のquirkドライバであれ、`struct hid_driver` を登録する。接続してきたデバイスとのマッチングは `.id_table`（`struct hid_device_id` の配列で、バス/ベンダー/製品でマッチし、`HID_USB_DEVICE()`、`HID_BLUETOOTH_DEVICE()`、`HID_I2C_DEVICE()` のようなヘルパーマクロで構築する）で行われる。マッチングはトランスポート層より上で行われるため、同じ構造体がトランスポートをまたいで機能する。

```c
// include/linux/hid.h
struct hid_driver {
	const char *name;
	const struct hid_device_id *id_table;

	struct list_head dyn_list;
	spinlock_t dyn_lock;

	bool (*match)(struct hid_device *dev, bool ignore_special_driver);
	int (*probe)(struct hid_device *dev, const struct hid_device_id *id);
	void (*remove)(struct hid_device *dev);

	const struct hid_report_id *report_table;
	int (*raw_event)(struct hid_device *hdev, struct hid_report *report,
			u8 *data, int size);
	const struct hid_usage_id *usage_table;
	int (*event)(struct hid_device *hdev, struct hid_field *field,
			struct hid_usage *usage, __s32 value);
	void (*report)(struct hid_device *hdev, struct hid_report *report);

	const __u8 *(*report_fixup)(struct hid_device *hdev, __u8 *buf,
			unsigned int *size);

	int (*input_mapping)(struct hid_device *hdev,
			struct hid_input *hidinput, struct hid_field *field,
			struct hid_usage *usage, unsigned long **bit, int *max);
	int (*input_mapped)(struct hid_device *hdev,
			struct hid_input *hidinput, struct hid_field *field,
			struct hid_usage *usage, unsigned long **bit, int *max);
	int (*input_configured)(struct hid_device *hdev,
				struct hid_input *hidinput);
	void (*feature_mapping)(struct hid_device *hdev,
			struct hid_field *field,
			struct hid_usage *usage);

	int (*suspend)(struct hid_device *hdev, pm_message_t message);
	int (*resume)(struct hid_device *hdev);
	int (*reset_resume)(struct hid_device *hdev);
	void (*on_hid_hw_open)(struct hid_device *hdev);
	void (*on_hid_hw_close)(struct hid_device *hdev);

/* private: */
	struct device_driver driver;
};
```

汎用のレポート処理を上書きするうえで重要なコールバックを、参照される順に示す。

- **`.report_fixup`** — `hid_open_report()` から、ディスクリプタがパースされるよりも前に一度だけ呼ばれる（上述）。生のディスクリプタのバイトにパッチを当てる。デバイスが明らかに誤ったディスクリプタ（低すぎる最大値、ディスクリプタの他の部分が示唆するものと一致しないUsage Pageのバイト）を出荷している場合に選ばれる手段である。
- **`.probe`** — ドライバ自身のprobeで、トランスポートのコアのprobeが `hid_device` をアタッチした後に呼ばれる。ほぼ必ず `hid_parse()` を呼び、次に `hid_hw_start()` を呼ぶ。
- **`.input_mapping`** — `hid-input.c` がusageをevdevイベントにどうマッピングするかを決めている間に、usageごとに一度呼ばれる（後述）。`1` を返すとマッピングを引き受けたことになる（ドライバがすでに自分で `hid_map_usage()` を呼んでいる）。`0` は汎用テーブルに任せる。負の値は「何もマッピングせず、このusageを完全に無視する」ことを意味する。
- **`.input_mapped`** — マッピング（汎用のものかドライバが提供したもの）が決まった*後*、`input_dev` のケーパビリティビットマップに確定される直前に、usageごとに一度呼ばれる。ドライバが、自分が作ったのではないマッピングを調整したり拒否したりできる。ここで負の値を返してもusageが抑止される。
- **`.raw_event`** — フィールドの抽出が行われる前に、届いたすべてのレポートについて、生のレポートのバイトを渡して呼ばれる。汎用処理の前にレポートを横取りする必要があるドライバや、汎用のフィールド抽出ロジックではまったく解釈できないバイトから入力イベントを合成したいドライバのための抜け道である。
- **`.event`** — 抽出されたフィールドの*値*ごとに一度、汎用のフィールド抽出の後、汎用の `hidinput_hid_event()` による `input_event()` への変換の前に（あるいはその代わりに）呼ばれる。

## トランスポートの階層化: `struct hid_ll_driver` {#transport-layering-struct-hidlldriver}

ここまで説明したもの（ディスクリプタのパース、`struct hid_device`、`hid-input.c` のマッピング）はどれも、レポートのバイトが実際にどうやってカーネルに届いたかを知らず、気にもしない。それは意図的に `struct hid_ll_driver`（「low-level driver」）として切り出されている。これは、トランスポートが一度実装し、そのトランスポート上のすべてのデバイスについてコアが呼び出すインターフェースである。

```c
// include/linux/hid.h
struct hid_ll_driver {
	int (*start)(struct hid_device *hdev);
	void (*stop)(struct hid_device *hdev);

	int (*open)(struct hid_device *hdev);
	void (*close)(struct hid_device *hdev);

	int (*power)(struct hid_device *hdev, int level);

	int (*parse)(struct hid_device *hdev);

	void (*request)(struct hid_device *hdev,
			struct hid_report *report, int reqtype);

	int (*wait)(struct hid_device *hdev);

	int (*raw_request) (struct hid_device *hdev, unsigned char reportnum,
			    __u8 *buf, size_t len, unsigned char rtype,
			    int reqtype);

	int (*output_report) (struct hid_device *hdev, __u8 *buf, size_t len);

	int (*idle)(struct hid_device *hdev, int report, int idle, int reqtype);
	bool (*may_wakeup)(struct hid_device *hdev);

	unsigned int max_buffer_size;
};
```

`.parse` は、トランスポートがデバイスから生のディスクリプタのバイトを読み取り（USBでは `GET_DESCRIPTOR` 制御リクエスト、Bluetoothではユーザー空間のSDP/HIDプロファイルのネゴシエーションによって接続確立時に渡されるバイト）、`hid_open_report()` が見つけられる場所にしまっておく処理である。それ*以降*のすべて（fixup、アイテムのパース、フィールドの抽出）は、トランスポートにかかわらず同一である。`.raw_request`/`.output_report` は、ドライバや入力イベントのパスがFeatureレポートやOutputレポート（LEDの状態、フォースフィードバック、ベンダーの設定データ）をデバイスに送り返す手段である。これも、呼び出しの形は1つで、その下のバイトがトランスポート固有になっている。

ツリー内にある実在の3つのトランスポートがこれを実装している。USB-HID（`drivers/hid/usbhid/hid-core.c`）が最初のもので、最も一般的である。

```c
// drivers/hid/usbhid/hid-core.c
static const struct hid_ll_driver usb_hid_driver = {
	.parse = usbhid_parse,
	.start = usbhid_start,
	.stop = usbhid_stop,
	.open = usbhid_open,
	.close = usbhid_close,
	.power = usbhid_power,
	.request = usbhid_request,
	.wait = usbhid_wait_io,
	.raw_request = usbhid_raw_request,
	.output_report = usbhid_output_report,
	.idle = usbhid_idle,
	.may_wakeup = usbhid_may_wakeup,
};
```

ここでの `.open`/`.close` は、USBの割り込みINエンドポイント（要求されずに送られてくるInputレポートを運ぶパイプ）と、存在する場合は割り込みOUTエンドポイントを接続する。`.raw_request` は、クラス固有の `SET_REPORT`/`GET_REPORT` リクエストとしてUSB制御パイプを通る。

Bluetooth-HIDはHID Profile（HIDP）の実装 `net/bluetooth/hidp/core.c` である。独自の `hid_ll_driver` を登録し、2つのL2CAPチャネル（制御チャネルと割り込みチャネル。USB-HIDが制御エンドポイントと割り込みエンドポイントから自然に得ている「ctrl」/「intr」の分割を模したもの）上でそれを駆動する。

```c
// net/bluetooth/hidp/core.c
static const struct hid_ll_driver hidp_hid_driver = {
	.parse = hidp_parse,
	.start = hidp_start,
	.stop = hidp_stop,
	.open  = hidp_open,
	.close = hidp_close,
	.raw_request = hidp_raw_request,
	.output_report = hidp_output_report,
};
```

カーネル自身のトランスポートのドキュメントは、この抽象化が対象として作られた実在の3つのトランスポートすべてを挙げている。ワイヤレベルでは「USB, I2C, Bluetooth-l2cap」であり、それらは「USB-HID, I2C-HID, BT-HIDP」ドライバを通じて表に出る（`Documentation/hid/hid-transport.rst`）。I2C-HID（`drivers/hid/i2c-hid/`）が3つ目で、ノートPCのタッチパッドやタッチスクリーンでよく使われる。BT-HIDPの2つの別々のL2CAPチャネルや、USB-HIDの別々の制御/割り込みエンドポイントの代わりに、単一のI2Cバストランザクションプロトコルを使う（I2C-HIDはMicrosoftが作成したクラスへの追加であり、元のUSB HID仕様の一部ではない）。ドライバの作者は、あるレポートのバイトを実際に運んだのがこれらのどれなのかを気にする必要がない。`struct hid_device->ll_driver` は、このデバイスをアタッチしたものを指しているだけである。

## evdevへのマッピング: `hid_map_usage()` と `hid-input.c` {#mapping-to-evdev-hidmapusage-and-hid-inputc}

パースが生み出すのは、生の `usage->hid` コードだけを持つ `struct hid_usage` で満たされた `struct hid_field` であり、`usage->type`/`usage->code` は未設定である。それを `evdev` が理解できるもの（`EV_KEY`/`KEY_A` のペアや `EV_ABS`/`ABS_X` のペア）に変換するのが `hid-input.c` の仕事であり、接続時に `hidinput_configure_usage()` でusageごとに一度行われる。マッピング自体は、小さな汎用のインラインヘルパーである。

```c
// include/linux/hid.h
static inline void hid_map_usage(struct hid_input *hidinput,
		struct hid_usage *usage, unsigned long **bit, int *max,
		__u8 type, unsigned int c)
{
	struct input_dev *input = hidinput->input;
	unsigned long *bmap = NULL;
	unsigned int limit = 0;

	switch (type) {
	case EV_ABS:
		bmap = input->absbit;
		limit = ABS_MAX;
		break;
	case EV_REL:
		bmap = input->relbit;
		limit = REL_MAX;
		break;
	case EV_KEY:
		bmap = input->keybit;
		limit = KEY_MAX;
		break;
	...
	}

	if (unlikely(c > limit || !bmap)) {
		pr_warn_ratelimited("%s: Invalid code %d type %d\n",
				    input->name, c, type);
		*bit = NULL;
		return;
	}

	usage->type = type;
	usage->code = c;
	*max = limit;
	*bit = bmap;
}
```

`hidinput_configure_usage()` はこれを呼ぶが、その前にドライバに先に判断する機会を与える。

```c
// drivers/hid/hid-input.c (hidinput_configure_usage(), abridged)
if (device->driver->input_mapping) {
	int ret = device->driver->input_mapping(device, hidinput, field,
			usage, &bit, &max);
	if (ret > 0)
		goto mapped;
	if (ret < 0)
		goto ignore;
}

switch (usage->hid & HID_USAGE_PAGE) {
case HID_UP_UNDEFINED:
	goto ignore;

case HID_UP_KEYBOARD:
	set_bit(EV_REP, input->evbit);

	if ((usage->hid & HID_USAGE) < 256) {
		if (!hid_keyboard[usage->hid & HID_USAGE]) goto ignore;
		map_key_clear(hid_keyboard[usage->hid & HID_USAGE]);
	} else
		map_key(KEY_UNKNOWN);

	break;
...
```

この `hid_keyboard[]` テーブルが汎用のKeyboardページのマッピングであり、usageの下位バイトでインデックスされる、Linuxキーコードの256エントリの平坦な配列である。インデックス4（usage `0x00070004`、Usage Page `Keyboard/Keypad`、Usage `0x04`。HID Usage Tablesによれば「Keyboard a and A」）には `30` が入っており、これは `KEY_A` である。この配列は文字どおり `drivers/hid/hid-input.c` の `hid_keyboard[]` であり、`hid_keyboard[4] == 30` がまさにこのマッピングで、普通のキーボードではドライバのコードはまったく関与しない。

マッピングが選ばれると（汎用のものであれ、ドライバが `.input_mapping` から自分で `hid_map_usage()` を呼んだものであれ）、確定される前にもう1つドライバのフックを通過する。

```c
// drivers/hid/hid-input.c (hidinput_configure_usage(), abridged)
mapped:
	if (!bit)
		return;

	if (device->driver->input_mapped &&
	    device->driver->input_mapped(device, hidinput, field, usage,
					 &bit, &max) < 0) {
		return;
	}

	set_bit(usage->type, input->evbit);
	...
```

実行時、デバイスがオープンされると、届いたすべてのレポートは `__hid_input_report()`（`drivers/hid/hid-core.c`）に到達する。この関数は、IDで対応する `struct hid_report` を見つけた後、HIDコアによるフィールド抽出の前に、ドライバの `.raw_event` にレポートのバイトを渡し、それから必ず汎用パスを実行する。

```c
// drivers/hid/hid-core.c (__hid_input_report(), abridged)
if (hdrv && hdrv->raw_event && hid_match_report(hid, report)) {
	ret = hdrv->raw_event(hid, report, data, size);
	if (ret < 0)
		goto unlock;
}

ret = hid_report_raw_event(hid, type, data, bufsize, size, interrupt);
```

`hid_report_raw_event()` はレポート全体を `hid_process_report()` に渡す。`hid_process_report()` は各フィールドのビットパックされた値を抽出し、レポート内のすべてのusageについて `hid_process_event()` を呼ぶ。この関数こそが、接続時に `hid_map_usage()` が設定した `type`/`code` のペアをここでようやく使って、`hidinput_hid_event()`、つまり `input_event(input, usage->type, usage->code, value)` 呼び出しを実際に行うものである。`hid_process_report()` がすべてのフィールドをディスパッチし終えた後で初めて、`hid_report_raw_event()` は `hidinput_report_event()` を呼ぶ。この関数は `hid->inputs` を走査して、アタッチされた各 `input_dev` に対して `input_sync()` を呼ぶ以外のことはしない。これが、`hid_process_event()` が今生成したイベントのバッチを締めくくる `EV_SYN`/`SYN_REPORT` の配布である。

## 実例: `hid-petalynx.c` {#worked-example-hid-petalynxc}

`drivers/hid/hid-petalynx.c` は、Petalynx Maxterリモコン用の、ツリー内にある実在の小さなquirkドライバで、全体で100行を少し超える程度の中で `.report_fixup` と `.input_mapping` の両方を示してくれる。まず、これが回避しているディスクリプタのバグである。デバイスのConsumerページの最大usage値が低すぎるため、パーサがディスクリプタの後のほうにある完全に正当なusageを拒否したり切り詰めたりしてしまう。

```c
// drivers/hid/hid-petalynx.c
/* Petalynx Maxter Remote has maximum for consumer page set too low */
static const __u8 *pl_report_fixup(struct hid_device *hdev, __u8 *rdesc,
		unsigned int *rsize)
{
	if (*rsize >= 62 && rdesc[39] == 0x2a && rdesc[40] == 0xf5 &&
			rdesc[41] == 0x00 && rdesc[59] == 0x26 &&
			rdesc[60] == 0xf9 && rdesc[61] == 0x00) {
		hid_info(hdev, "fixing up Petalynx Maxter Remote report descriptor\n");
		rdesc[60] = 0xfa;
		rdesc[40] = 0xfa;
	}
	return rdesc;
}
```

パッチを当てる前の防御的なバイトパターンのチェックに注意してほしい。`report_fixup` はこのドライバの `id_table` にマッチするすべてのデバイスで実行されるため、最大usage値のアイテムの2バイトを上書きする前に、期待する正確なバイトが実際にそこにあることを確認している（「同じ」デバイスでもファームウェアのリビジョンが異なれば、別の、すでに正しいディスクリプタを持っている可能性がある）。

次に、ドライバはベンダー/Logitech関連の2つのusageページを、汎用テーブルが知らない標準的なLinuxキーコードに再マッピングする。リモコンの色付きボタン用の「LogiVendor」ページと、このデバイスがHID Usage Tables自身が推奨するマッピングには従っていないらしい、次へ/戻る用の2つのConsumerページのusageである。

```c
// drivers/hid/hid-petalynx.c
#define pl_map_key_clear(c)	hid_map_usage_clear(hi, usage, bit, max, \
					EV_KEY, (c))
static int pl_input_mapping(struct hid_device *hdev, struct hid_input *hi,
		struct hid_field *field, struct hid_usage *usage,
		unsigned long **bit, int *max)
{
	if ((usage->hid & HID_USAGE_PAGE) == HID_UP_LOGIVENDOR) {
		switch (usage->hid & HID_USAGE) {
		case 0x05a: pl_map_key_clear(KEY_TEXT);		break;
		case 0x05b: pl_map_key_clear(KEY_RED);		break;
		case 0x05c: pl_map_key_clear(KEY_GREEN);	break;
		case 0x05d: pl_map_key_clear(KEY_YELLOW);	break;
		case 0x05e: pl_map_key_clear(KEY_BLUE);		break;
		default:
			return 0;
		}
		return 1;
	}

	if ((usage->hid & HID_USAGE_PAGE) == HID_UP_CONSUMER) {
		switch (usage->hid & HID_USAGE) {
		case 0x0f6: pl_map_key_clear(KEY_NEXT);		break;
		case 0x0fa: pl_map_key_clear(KEY_BACK);		break;
		default:
			return 0;
		}
		return 1;
	}

	return 0;
}
```

`pl_map_key_clear()` 呼び出しの成功後に `1` を返すと、`hidinput_configure_usage()` にマッピングはすでに完了したと伝わり、このusageについては汎用のswitchが丸ごとスキップされる。この2つのページ上で、列挙された5つ/2つのusage以外のものについては `return 0` に落ちることで、usageを黙って捨てるのではなく、汎用テーブルに試させる。

そして、両方をドライバに結び付ける配管である。

```c
// drivers/hid/hid-petalynx.c
static int pl_probe(struct hid_device *hdev, const struct hid_device_id *id)
{
	int ret;

	hdev->quirks |= HID_QUIRK_NOGET;

	ret = hid_parse(hdev);
	if (ret) {
		hid_err(hdev, "parse failed\n");
		goto err_free;
	}

	ret = hid_hw_start(hdev, HID_CONNECT_DEFAULT);
	if (ret) {
		hid_err(hdev, "hw start failed\n");
		goto err_free;
	}

	return 0;
err_free:
	return ret;
}

static const struct hid_device_id pl_devices[] = {
	{ HID_USB_DEVICE(USB_VENDOR_ID_PETALYNX, USB_DEVICE_ID_PETALYNX_MAXTER_REMOTE) },
	{ }
};
MODULE_DEVICE_TABLE(hid, pl_devices);

static struct hid_driver pl_driver = {
	.name = "petalynx",
	.id_table = pl_devices,
	.report_fixup = pl_report_fixup,
	.input_mapping = pl_input_mapping,
	.probe = pl_probe,
};
module_hid_driver(pl_driver);
```

ここでの `.probe` は、HIDドライバに必要な最小限に近い。`hid_parse()`（`.report_fixup` が実際に呼び出されるのは、この中の `hid_open_report()` である）、次に `hid_hw_start(hdev, HID_CONNECT_DEFAULT)` である。このフラグの組み合わせ（`HID_CONNECT_HIDINPUT|HID_CONNECT_HIDRAW|HID_CONNECT_HIDDEV|HID_CONNECT_FF`）は、デバイスをevdev、`/dev/hidraw*`、レガシーなhiddevインターフェース、フォースフィードバックに一度に接続する。それ以外のすべて（レポートIDのルックアップテーブル、`struct hid_field`、このドライバが明示的には一度も作成しないevdevデバイス）は、fixup済みのディスクリプタとドライバの2つのコールバックを使って、汎用のコアが構築する。

## HID-BPF: 新しいドライバなしでのfixup {#hid-bpf-fixups-without-a-new-driver}

`.report_fixup` と `.input_mapping` はどちらも、カーネルモジュールを書き、ビルドし、アップストリームに取り込んでもらう必要がある。1つのデバイスのバグ修正にしては遅い道のりである。HID-BPF（`Documentation/hid/hid-bpf.rst`、`CONFIG_HID_BPF`）は、より新しい補完的な仕組みである。実行時にユーザー空間からロードされるeBPFプログラムを特定のデバイスにアタッチして、レポートディスクリプタをfixupしたりレポートイベントを横取りしたりでき、新しいドライバがツリーに入る必要はまったくない。`struct hid_device` の `bpf_rdesc`/`bpf_rsize` フィールド（上で示したように、`hid_open_report()` がパースの実際の出発点として使う）は、まさにBPFによるディスクリプタのfixupが、通常のドライバマッチングのパスにある `.report_fixup` に出番が来るよりも*前に*実行されるようにするために存在する。カーネル自身のHID-BPFのドキュメントは、その動機を率直に述べている。レポートディスクリプタの「one key or one byte」を直すためだけに存在するドライバという一般的なケースでは、従来の道は「require[s] a kernel patch and the subsequent shepherding into a release, a long and painful process for users」（カーネルパッチと、その後リリースへと導く手間を必要とし、ユーザーにとって長く苦痛なプロセスである）のに対し、eBPFプログラムならユーザーが検証して直接ロードでき、カーネルのリリースを待つ必要がない（`Documentation/hid/hid-bpf.rst`）。Benjamin Tissoiresが2022年にこの仕組みをアップストリームに提案しており、そのパッチシリーズのv2のカバーレターがLWNにアーカイブされている（[Introduce eBPF support for HID devices](https://lwn.net/Articles/886860/)）。状態を持つものや複雑なものについては `.input_mapping`/`.raw_event` の代わりにはならず、それらには今も本物のドライバが必要である。しかし、「このディスクリプタの1バイトが間違っている」という狭く極めて一般的なケースについては、アップストリームが一般にまったく新しいquirkモジュールよりも好む、より手間の少ない選択肢となっている。

## デバイスからユーザー空間までの全経路 {#the-full-path-device-to-userspace}

```
 Physical device
 (keyboard, mouse, gamepad, touchpad...)
        │  descriptor bytes + report bytes, over one of:
        ▼
 ┌─────────────────┬─────────────────┬─────────────────┐
 │  USB            │  Bluetooth      │  I2C            │
 │ (interrupt IN/  │ (HIDP, two      │ (I2C-HID        │
 │ OUT endpoints)  │ L2CAP chans)    │ protocol)       │
 └────────┬────────┴────────┬────────┴────────┬────────┘
          ▼                 ▼                 ▼
         usb_hid_driver    hidp_hid_driver   i2c_hid_ll_driver
          │  each is a struct hid_ll_driver: .parse/.start/.open/
          │  .raw_request/.output_report/...
          └─────────────────┬─────────────────┘
                            ▼
                HID core (drivers/hid/hid-core.c)
              ┌──────────────────────────────────────┐
              │ hid_open_report()                    │
              │   .report_fixup (if any)             │  ← quirk driver, or HID-BPF
              │   hid_parse_collections()            │     rdesc fixup program
              │     → struct hid_report/hid_field/   │
              │       hid_usage/hid_collection       │
              │                                      │
              │ __hid_input_report() (per packet)    │
              │   .raw_event (if any)                │  ← quirk driver, raw bytes
              │   hid_report_raw_event()             │
              │     → field value extraction         │
              └──────────────────┬───────────────────┘
                                 ▼
                hid-input.c: hidinput_configure_usage()
              ┌───────────────────────────────────────────────────────┐
              │ .input_mapping (if any)  ──1──► claims mapping──┐     │
              │       │0                                        │     │
              │       ▼                                         │     │
              │ generic switch(usage->hid                       │     │
              │   & HID_USAGE_PAGE) tables                      │     │
              │       │                                         │     │
              │       ▼                                         │     │
              │ hid_map_usage(): sets                           │     │
              │   usage->type / usage->code                     │     │
              │       │                                         │     │
              │       ▼                                         │     │
              │ .input_mapped (if any) ◄────────────────────────┘     │
              │   may veto/adjust                                     │
              └───────────────────────────┬───────────────────────────┘
                                          ▼
              hidinput_hid_event() → input_event(dev,
                usage->type, usage->code, value)
                                          ▼
                      evdev (/dev/input/eventN)
                                          ▼
                      userspace (libinput, Xorg, Wayland
                      compositor, a game reading /dev/input
                      directly...)
```

## 参考文献 {#further-reading}

### カーネルソース {#kernel-source}

- [`include/linux/hid.h`](https://raw.githubusercontent.com/torvalds/linux/master/include/linux/hid.h) — `struct hid_device`、`struct hid_driver`、`struct hid_ll_driver`、`struct hid_report`/`hid_field`/`hid_usage`/`hid_collection`、`hid_map_usage()`
- [`include/uapi/linux/hid.h`](https://raw.githubusercontent.com/torvalds/linux/master/include/uapi/linux/hid.h) — `enum hid_report_type`（`HID_INPUT_REPORT`/`HID_OUTPUT_REPORT`/`HID_FEATURE_REPORT`）
- [`drivers/hid/hid-core.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/hid/hid-core.c) — `hid_open_report()`、`hid_parse_collections()`、`__hid_input_report()`、`hid_report_raw_event()`
- [`drivers/hid/hid-input.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/hid/hid-input.c) — `hidinput_configure_usage()`、`hid_keyboard[]`、`hidinput_hid_event()`
- [`drivers/hid/usbhid/hid-core.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/hid/usbhid/hid-core.c) — `usb_hid_driver`、USB-HIDの `hid_ll_driver`
- [`net/bluetooth/hidp/core.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/bluetooth/hidp/core.c) — `hidp_hid_driver`、BT-HIDPの `hid_ll_driver`
- [`drivers/hid/hid-petalynx.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/hid/hid-petalynx.c) — 上の実例のquirkドライバ
- [`drivers/hid/hid-generic.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/hid/hid-generic.c) — 普通のマウスやキーボードが使う、quirkコールバックをまったく持たない汎用ドライバ

### 関連ページ {#related-pages}

- [入力サブシステム: evdevと入力コア](README.md) — HIDの `hid_map_usage()` と `input_event()` 呼び出しが供給する先の `input_dev`/`evdev` 層
- [入力サブシステムの事件簿](war-stories.md) — このページのパースとマッピングの仕組みの上に位置する、入力/HIDスタックの事件

### 外部資料 {#external}

- [Introduction to HID report descriptors](https://docs.kernel.org/hid/hidintro.html) — usageページ、usage、レポート、コレクションをめぐるカーネル自身のガイドで、マウスのディスクリプタの実例付き
- [HID I/O Transport Drivers](https://docs.kernel.org/hid/hid-transport.html) — `hid_ll_driver` の契約と、USB-HID/BT-HIDP/I2C-HIDのトランスポート
- [HID-BPF](https://docs.kernel.org/hid/hid-bpf.html) — ディスクリプタのfixupとイベントのフィルタリングのための、完全なquirkドライバに代わるeBPFベースの手段
- [Introduce eBPF support for HID devices](https://lwn.net/Articles/886860/) — HID-BPFをアップストリームに提案したBenjamin Tissoiresのv2カバーレター。LWNにアーカイブされている（LWNの編集記事ではなく、メーリングリストへの投稿）
