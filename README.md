![](https://github.com/OldDriver007/Wallet/blob/master/icon_logo.png) 




# AToken Android客服端 重点介绍




# About/关于
```
AToken是一款易用的数字资产钱包，可以存储比特币(BTC)、莱特币(LTC)、以太坊(ETH)、以太经典(ETC)、柚子(EOS)等的多链轻钱包
```



#### 最新版本

日期|version|targetSdkVersion|buildToolsVersion
---|---|---|---
2018/12|&ensp;2.5.3|&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;28|&ensp;&ensp;&ensp;&ensp;&ensp;28.0.3

#### GooglePlay
https://play.google.com/store/apps/details?id=wallet.gem.com


-------------------------------

# 目录
+ [AToken支持币种介绍](#1)
+ [创建和导入钱包](#2)
+ [数据库表结构和多钱包管理](#3)
+ [EOS账户系统及交易构造](#4)
+ [以太坊交易构造](#5)
+ [DAPP](#6)
+ [打包和发版注意事项](#7)



------------------------------
#### <h2 id="1"> 一、AToken支持币种介绍</h2>
![](https://github.com/OldDriver007/Wallet/blob/master/screenshot.png)  


主链币：BTC ETH EOS LTC DOGE  
分叉币：BCH BTG BCD SBTC  
Token:  ETH ERC20  EOS Token  


#### <h2 id="2"> 一、创建和导入钱包</h2>

创建和导入钱包流程主要是，先获取随机数，创建钱包是利用java API生成，导入钱包是用用户输入的助记词推导出随机种子熵，之后传给HD-Wallet,根据不同的币种生成相应私钥，公钥和地址，公钥，币种地址保存在本地数据库，私钥需用户输入密码推导，同时币种地址需提交  给后台服务入库。助记词是用来管理各个币种私钥的，一个钱包对应一套助记词，相当于银行卡和密码，丢失后无法找回。



#### 钱包遵循BIP44协议  
m / purpose' / coin_type' / account' / change / address_index  
': 固定值44', 代表是BIP44  
coin_type': 这个代表的是币种, 可以兼容很多种币, 比如BTC是0', ETH是60',EOS是194'  
BTC  m/44'/0'/0'/0/0  
ETH  m/44'/60'/0'/0/0  
EOS  m/44'/194'/0'/0/0  
```
  private DeterministicKey getDiffAccount(DeterministicKey master, int d) {
        switch (d) {//0-btc,2-ltc,3-doge,60-eth、etc 194-eos
            case 0:
                return getAccount(master, 44, 0, 0);
            case 2:
                return getAccount(master, 44, 2, 0);
            case 3:
                return getAccount(master, 44, 3, 0);
            case 60:
                return getAccount(master, 44, 60, 0);
            case 70:
                return getAccount(master, 44, 60, 1);
            case 194:
                return getAccount(master, 44, 194, 0);
            default:
                return null;
        }
    }

```

其中BTC在生成时创建了100个地址，EOS是两个公钥




#### 导入钱包流程
![导入助记词时许图](https://github.com/OldDriver007/Wallet/blob/master/SequenceDiagram1.png)

#### <h2 id="3">二、数据库表结构和多钱包管理</h2>

![表结构](https://github.com/OldDriver007/Wallet/blob/master/db.png)

#### 数据库
tx.db
 1. hd_account_addresses&ensp;&ensp;地址表&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;主键 walletId
 2. wallet&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;钱包表&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;主键 walletId
 3. eos_account&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;EOS账户表&ensp;&ensp;主键 walletId

address.db
1.hd_account HD表


![hd_account_addresses表结构](https://github.com/OldDriver007/Wallet/blob/master/address.png)

```
  public static final String CREATE_HD_ACCOUNT_ADDRESSES = "create table if not exists " +
            "hd_account_addresses " +  
            "(hd_account_id integer not null" +  //coinType
            ", path_type integer not null" + 
            ", address_index integer not null" +  //地址索引
            ", is_issued integer not null" +
            ", address text not null" +           //地址
            ", pub text not null" +               //公钥
            ", is_synced integer not null" +
            ", wallet_id text " +                 //钱包ID
            ", primary key (address));";
```



###如何获取地址或公钥
通过hd_account_id ,address_index和 wallet_id可以定位到地址和公钥


```
    private static String getETHAddress() {
        String walletId = ManagerWalletDBHelper.SingleTon.getInstance().getCurrentWalletId();
        int index = ManagerWalletDBHelper.SingleTon.getInstance().getCoinIndexByWalletId("ETH");
        String mAddr = HDAccountAddressProvider.getInstance().getExternalAddr(CoinController.SingleTon.getInstance().getCoinIntByName("eth"), index, walletId);

        if (TextUtils.isEmpty(mAddr)) {

        }
        return mAddr;
    }

    private static String getEosOwerAddress() {
        String walletId = ManagerWalletDBHelper.SingleTon.getInstance().getCurrentWalletId();
        String mAddr = HDAccountAddressProvider.getInstance().getExternalAddr(CoinController.SingleTon.getInstance().getCoinIntByName("EOS"), 0, walletId);
        return mAddr;
    }
   
```
#### 如何给新币种添加地址
需要用户输入密码
```   
/**
     * 用于导入其他hd模型的主链币
     * 0-btc,2-ltc,3-doge,60-eth和etc
     *
     * @param password
     * @param hdaccountId
     * @return
     */
    public static Observable<Boolean> importCoin(String password, int hdaccountId) {
        return Observable.just(password)
                .map(pwd -> {
                    try {
                        String hdseed = AESUtils.decryptData(AddressGenerateUtils.getPwdHash(pwd), SP.get(Constant
                                .SP_SEED, ""));
                        HDAccount account = new HDAccount(HexUtils.hexStringToBytes(hdseed), false, hdaccountId);
                        return account;
                    } catch (Exception e) {
                        return null;
                    }
                })
                .doOnNext(account -> {
                    if (account != null) {
                        try {
                            account.wipe();
                        } catch (Exception e) {
                            throw Exceptions.propagate(e);
                        }
                    }
                })
                .map(account -> account != null);
    }
```
### wallet表

```
    public static final String CREATE_WALLET = "create table wallet " +
            "(wallet_id TEXT not null , " +      //wallet Id
            "wallet_name TEXT not null , " +     //钱包名
            "wallet_random_seed TEXT not null , " +  //熵
            "wallet_seed TEXT not null , " +        
            "wallet_password_seed TEXT not null , " + //密钥种子
            "wallet_phone TEXT not null , " +         //绑定到手机号
            "wallet_mnemonic_saved TEXT not null , " + //助记词是否备份
            "wallet_coin_index TEXT not null , " +     //BTC币种索引
            "isDefaultWallet integer not null , " +    //是否为主钱包
            "assets TEXT not null , " +                //钱包总资产缓存
            "eos_account TEXT not null , " +           //弃用
            "eos_default_account TEXT not null , " +    //弃用
            "eos_seed TEXT not null , " +                //弃用
            "eos_ower_pub TEXT not null , " +           //弃用
            "eos_active_pub TEXT not null , " +         //弃用
            "eos_other TEXT not null );";             //钱包缓存数据
``` 
### 多钱包如何切换

```
 
    private void switchWallet(String amount) {
        Observable.just(1)
                .map(i -> {
                    boolean success = false;
                    try {
                        HDAccount.MultiWalletModel multiWalletModel;
                        String walletId = ManagerWalletDBHelper.SingleTon.getInstance().getCurrentWalletId();
                        multiWalletModel = ManagerWalletDBHelper.SingleTon.getInstance().queryWalletDataById(walletId);
                        if(!TextUtils.isEmpty(multiWalletModel.getWallet_phone())) {
                            SP.set(Constant.PHONE, multiWalletModel.getWallet_phone());
                        }
                        SP.set(Constant.SP_SEED, multiWalletModel.getWallet_seed());
                        SP.set(Constant.SP_RANDOM_SEED, multiWalletModel.getWallet_random_seed());
                        SP.set(Constant.SP_PASSWORD_SEED, multiWalletModel.getWallet_password_seed());
                        boolean isBackup = false;
                        if(multiWalletModel.getWallet_mnemonic_saved().equals("true")){
                            isBackup = true;
                        }
                        ManagerWalletDBHelper.SingleTon.getInstance().setCurrentWalletBackupState(isBackup);

//                        for (String coinName : CoinController.SingleTon.getInstance().getDefaultList()) {
//                            SP.set(Constant.SP_COIN_INDEX + coinName.toUpperCase(), 0);
//                        }
                        success = true;
                    } catch (Exception e) {
                        success = false;
                    } finally {
                        return success;
                    }
                }).compose(GRetrofit.observeOnMainThread(getUI(), Schedulers.computation()))
                .subscribe(new EasySubscriber<Boolean>() {

                    @Override
                    public void onStart() {
                        super.onStart();
                       // showLoading();
                    }
                    @Override
                    public void onNext(Boolean aBoolean) {
                        super.onNext(aBoolean);
                        finishLoading();
                         SP.set(Constant.SWITCH_WALLET_FOR_SETTING, isFromSetting);
                         if(isFromSetting){
                             Intent intent = new Intent();
                             getActivity().setResult(16, intent);
                             getActivity().finish();
                         }else {
                            getActivity().finish();
                        }
                    }
                });


    }
```

### EOS表

目前一个wallet对应一个EOS账户
```
 public static final String  CREATE_EOS_ACCOUONT_TABLE  = "create table " + EOS_TABLLE_NAME  +
            "( " + WALLET_ID + " TEXT not null , " +  //钱包id
            BALANCE + " TEXT not null , " +
            EOS_ACCOUNT_LIST +" TEXT not null , " +  //账户详情
            EOS_MAIN_ACCOUNT +" TEXT not null , " +  //主账户
            EOS_OWER_PUB +" TEXT not null , " +
            EOS_ACTIVE_PUB +" TEXT not null , " +
            EOS_OWER_PRIV_ENCRY +" TEXT not null , " +
            EOS_ACTIVE_PRIV_ENCRY +" TEXT not null , " +
            EOS_ETH_PRIV_ENCRY +" TEXT not null , " +
            EOS_ACCOUNT_STATE +" TEXT not null , " +   //账户权限状态
            EOS_REGISTERED_WALLET +" TEXT not null , " +
            EOS_OTHER +" TEXT not null );";

```
### 升级数据库注意事项

```

public class TxDatabaseHelper extends SQLiteOpenHelper {

    public static final int DB_VERSION = 4;
    private static final String DB_NAME = "tx.db";
    String noname = null;
    public TxDatabaseHelper(Context context) {
        super(context, DB_NAME, null, DB_VERSION);
         noname = context.getResources().getString(R.string.string_no_name);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        createBlocksTable(db);
        createTxsTable(db);
        createAddressTxsTable(db);
        createInsTable(db);
        createOutsTable(db);
        createPeersTable(db);
        createHDAccountAddress(db);
        createWalletTable(db);
        createEosAccountTable(db);
        db.execSQL(AbstractDb.CREATE_HD_ACCOUNT_ACCOUNT_ID_AND_PATH_TYPE_INDEX);

    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {

        switch (oldVersion) {
            case 1:
                createWalletTableDB1(db);
                db.execSQL("ALTER TABLE hd_account_addresses ADD COLUMN wallet_id text");
                db.insert("wallet", null, ManagerWalletDBHelper.SingleTon.getInstance().insertOldWalletDB(noname));
                ContentValues values =  new ContentValues();
                values .put("wallet_id", ManagerWalletDBHelper.SingleTon.getInstance().getCurrentWalletId());
                db.update("hd_account_addresses",values,null,null);
            case 2:
                db.execSQL("ALTER TABLE wallet ADD COLUMN assets text");
                db.execSQL("ALTER TABLE wallet ADD COLUMN eos_account text");
                db.execSQL("ALTER TABLE wallet ADD COLUMN eos_default_account text");
                db.execSQL("ALTER TABLE wallet ADD COLUMN eos_seed text");
                db.execSQL("ALTER TABLE wallet ADD COLUMN eos_ower_pub");
                db.execSQL("ALTER TABLE wallet ADD COLUMN eos_active_pub");
                db.execSQL("ALTER TABLE wallet ADD COLUMN eos_other");
                saveOldWalletCoinIndex(db);
            case 3:
                createEosAccountTable(db);
                insertOldEosDB(db);
            default:
                break;
        }



    }
```
####   注意 删钱包要删除以上关联到表还应删除hd_account


#### <h2 id="4"> 一、EOS账户系统及交易构造</h2>

#### EOS账户系统
EOS到账户由12位到数字字母组成，可以是全数字，数字必须是1-5，字母a-z需小写组成，通过节点可以查询到账户的一些基本信息，
owner公钥和active公钥，CPU,NET,RAM使用情况，赎回日期

```
{
  "account_name": "gy4tsmjvhege",
  "head_block_num": 31490345,
  "head_block_time": "2018-12-11T06:27:42.000",
  "privileged": false,
  "last_code_update": "1970-01-01T00:00:00.000",
  "created": "2018-06-09T12:24:03.500",
  "core_liquid_balance": "6.2229 EOS",
  "ram_quota": 12235,
  "net_weight": 18776,
  "cpu_weight": 138382,
  "net_limit": {
    "used": 909,
    "available": 1480423,
    "max": 1481332
  },
  "cpu_limit": {
    "used": 9341,
    "available": 0,
    "max": 6661
  },
  "ram_usage": 8166,
  "permissions": [
    {
      "perm_name": "active",
      "parent": "owner",
      "required_auth": {
        "threshold": 1,
        "keys": [
          {
            "key": "EOS567xSwyAG9e6HmQf1heYRe5riF9GhEov8XV4QgTefHrD9vjyRh",
            "weight": 1
          }
        ],
        "accounts": [],
        "waits": []
      }
    },
    {
      "perm_name": "owner",
      "parent": "",
      "required_auth": {
        "threshold": 1,
        "keys": [
          {
            "key": "EOS567xSwyAG9e6HmQf1heYRe5riF9GhEov8XV4QgTefHrD9vjyRh",
            "weight": 1
          }
        ],
        "accounts": [],
        "waits": []
      }
    }
  ],
  "total_resources": {
    "owner": "gy4tsmjvhege",
    "net_weight": "1.8776 EOS",
    "cpu_weight": "13.8382 EOS",
    "ram_bytes": 10835
  },
  "self_delegated_bandwidth": {
    "from": "gy4tsmjvhege",
    "to": "gy4tsmjvhege",
    "net_weight": "1.7276 EOS",
    "cpu_weight": "13.1382 EOS"
  },
  "refund_request": null,
  "voter_info": {
    "owner": "gy4tsmjvhege",
    "proxy": "",
    "producers": [
      "eosflytomars"
    ],
    "staked": 148658,
    "last_vote_weight": "76907583358.25599670410156250",
    "proxied_vote_weight": "0.00000000000000000",
    "is_proxy": 0
  }
}
```

#### 什么是owner权限和active权限
EOS一个账户两种类型公钥，分别是owner公钥和active公钥，公钥可以变更或添加多个，同时私钥对应各自公钥，一个公钥也可以创建多个账户，所以在用户导入EOS私钥或激活EOS钱包时需通过公钥查询账户列表，owner相当于账户所有者，可以称之为拥有者权限，权限最高，可以管理active，active又称管理者权限，用于交易，投票等日常操作。owner公钥和active公钥也可以相同，AToken在新创建账户时默认owner和active两个公钥，导入私钥时可以单独导一个私钥，也可以导入私钥对，但需要注意的是导入是要存储账户权限类型，在签名交易时要根据权限类型来签名，否则签名不成功

#### 如何获取EOS权限类型
```
    public static String getPermissionsType(String account) {
        int permissionsType = EosAccountDBHelper.getInstance().queryEosAccountPermissionstype(account);
        String owerOrAcitve = "active";
        switch (permissionsType) {
            case EosAccountDBHelper.OWER:
                owerOrAcitve = "owner";
                break;
            case EosAccountDBHelper.ACTIVE:
                owerOrAcitve = "active";
                break;
            case EosAccountDBHelper.OWER_ACTIVE:
                owerOrAcitve = "active";
                break;
            default:
                owerOrAcitve = "active";
                break;

        }
        return owerOrAcitve;
    }
```

#### 什么是映射用户，什么是194用户
EOS之前是ETH的代币，在2018年6月主网上线，变更为主链，原持有EOS代币的用户需拿ETH的公钥传给EOS主网，保证升级后币不会丢失，是为映射，也称60用户映射后会分配一个12位的账户，如果是新创建的EOS账户，BIP44里EOS Id为194,是位194用户，当创建账户时首先得判断60公钥是否包含EOS账户，如果有就是映射用户

#### EOS创建账户前的时序图
EOS任何行为都是交易，创建账户也是，所以新用户是无法创建的只能是有账户的帮助创建，创建预先分配4kRAM,0.1EOSCPU和NET，目前看来CPU0.1个是完全不够的
![](https://github.com/OldDriver007/Wallet/blob/master/creatAccount1.png) 

#### EOS交易构造

#### createAccountAndSign()&ensp;&ensp;账户
#### makeEosTranscation()&ensp;&ensp;/转账
#### doEosAction()&ensp;&ensp;/执行 投票，抵押，赎回，购买内存，赎回后退款,出售内存









 



1. 沟通和交流


    QQ 交流群1
    
        

  
