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
### EOS表

目前一个wallet对应一个EOS账户
```
 public static final String  CREATE_EOS_ACCOUONT_TABLE  = "create table " + EOS_TABLLE_NAME  +
            "( " + WALLET_ID + " TEXT not null , " +  //钱包id
            BALANCE + " TEXT not null , " +
            EOS_ACCOUNT_LIST +" TEXT not null , " +  //存储账户所有信息
            EOS_MAIN_ACCOUNT +" TEXT not null , " +  //主账户
            EOS_OWER_PUB +" TEXT not null , " +     //未用
            EOS_ACTIVE_PUB +" TEXT not null , " +    //未用
            EOS_OWER_PRIV_ENCRY +" TEXT not null , " +  //未用
            EOS_ACTIVE_PRIV_ENCRY +" TEXT not null , " +  //未用
            EOS_ETH_PRIV_ENCRY +" TEXT not null , " +   //未用
            EOS_ACCOUNT_STATE +" TEXT not null , " +   //账户权限状态
            EOS_REGISTERED_WALLET +" TEXT not null , " + //未用
            EOS_OTHER +" TEXT not null );";            //未用

```

```
public class EosAccountListBean implements Serializable {


    private String eosAccountName; //账户名
    private String balance;        //余额
    private int isMainAccount;      //是否是主账户
    private int isRegistered_Wallet; //是否绑定walletId
    private String owrPublicKey;     //owner公钥
    private String activePublicKey;   //actvie公钥
    private String owrPrivateKey;     //如果是导入私钥用户，加密存储的owner私钥
    private String activePrivate;  //如果是导入私钥用户，加密存储的active私钥
    private int permissionsType;   //权限类型
    private int owerWeight;        //owner权重
    private int activeWeight;      //owner权重
    
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

#### EOS激活账户时序图
EOS任何行为都是交易，创建账户也是，所以新用户是无法创建的只能是有账户的帮助创建，创建预先分配4kRAM,0.1EOSCPU和NET，目前看来CPU0.1个是完全不够的
激活账户实质是通过公钥来查询是否有EOS账户，首先查找是否是映射用户，再次查找是否是194用户，然后把公钥插入数据库，并把账户绑定到walletId
![](https://github.com/OldDriver007/Wallet/blob/master/creatAccount1.png) 

#### EOS交易构造

1.createAccountAndSign()&ensp;&ensp;创建账户  
2.makeEosTranscation()&ensp;&ensp;转账  
3.doEosAction()&ensp;&ensp;执行 投票，抵押，赎回，购买内存，赎回后退款,出售内存  


交易参数

```
       EOSTranscationManager.EosTransferParams params = new EOSTranscationManager.EosTransferParams()
                .activity(this)
                .amount(amount)       //金额 EOS金额必须是小数点后四位，如果没小数则补零，有Token的精度不是四位，需要根据精度来补位
                .conment(conment)     //备注memo
                .from(fromAddress)    //转账地址
                .to(toAddress)        //收款地址
                .tokenName(tokenName) //币种类型 EOS传EOS,Token传Token Name
                .contract(isEosToken ? tokenFullName :EOSUtils.EOS_TOKEN_ACCOUNT) //合约，Token交易用Token的合约
                .pwd(pwd)             //密码
                .ui(getUI());

        EOSTranscationManager.SingleTon.getInstance().makeEosTranscation(params);
```        


![交易时序图](https://github.com/OldDriver007/Wallet/blob/master/eos_tx.png)

### getEosInfo   节点基本信息，交易时要用到headBlockId,chainId
```
	"head_block_time": "2018-12-12T02:58:05.500",
    			 "block_cpu_limit": 80270,
    			 "chain_id": "aca376f206b8fc25a6ed44dbdc66547c36c6c33e3a119ffbeaef943642f0e906",
    			 "head_block_producer": "eosliquideos",
    			 "head_block_id": "01e2c08186b61c1fe6f4475343de80ca22698ace3bf047fa068a8181d54fbaa0",
    			 "head_block_num": 31637633,
    			 "virtual_block_cpu_limit": 685819,
    			 "virtual_block_net_limit": 1048576000,
    			 "server_version_string": "mainnet-1.3.2",
    			 "block_net_limit": 1044144,
    			 "last_irreversible_block_num": 31637298,
    			 "server_version": "9e62e735",
    			 "last_irreversible_block_id": "01e2bf32ed077fc1d7be4329cc4401affd119e918318399ebd0cd9184dd9c334"}
```

### EosTransfer 
  AToken为了安全，编码是客户端完成的，防止服务端被攻击篡改交易数据，所以用EosTransfer对交易参数进行编码序列化，编码参数转账地址，金额，备注等 
  编码分为     
  String&nbsp;&nbsp;  对备注编码
  TypeName&nbsp;&nbsp; 对账号编码
  TypeAsset&nbsp;&nbsp; 对金额币种编码 
### Action
   用来执行操作，如合约，转账  ,Action可以一次执行多个，如创建账户执行来三个购买内存，创建账户，抵押资源三个action
   
   ```
    Actions actions = new Actions();
        actions.setAuthorization(permissons);   //私钥权限 owner还是active
        actions.setData(hexdata);               //参数编码
        actions.setAccount(new TypeName(contract)); //合约
        actions.setName(new TypeName(actionName));  //action name  如交易：transfer
   
   ```

### EosTransaction 序列化后ECC签名

如何获取active私钥 owner私钥

 ```
      String  pri_key =  EOSUtils.getEosPrivateKey(params.from,params.pwd,EosAccountDBHelper.OWER);
      String  active_key = EOSUtils.getEosPrivateKey(params.from,params.pwd,EosAccountDBHelper.ACTIVE); 
 ```

签名

```
    public String sign(String pri_key, TypeChainId cid) throws Exception {
        byte[] packed = getDigestForSignature(cid);
        DebugLog.d(HexUtils.getHex(packed));

        EcSignature s1 = EcDsa.sign(Sha256.from(packed),new EosPrivateKey(pri_key));
        DebugLog.d("sig2=="+s1.toString());

        return s1.toString();
    }
```
### pushEosTransaction 广播交易








#### EOS资源管理

EOS资源包括RAM(内存)， NET(网络带宽)，CPU(CPU带宽)
EOS进行转账，抵押，赎回，投票，执行合约要消耗RAM,需要花费EOS来购买，RAM价格会波动，自己可以购买出售，也可以帮别人购买，不能帮别人出售
NET带宽和CPU带宽取决于过去三天消费的平均值，防止过多交易被攻击，转账要消耗这些带宽，单位时间内交易次数越多消耗越多，会造成抵押多少都不够都问题，随着时间推移，会自动释放，NETCPU需要通过抵押EOS来获取，也可以赎回，因抵押也是交易，所以会造成CPU不够无法抵押的现象，这时候需要别人帮助抵押，抵押分为租赁和过户,给别人抵押可 0 和1, 0代表租借 1过户，给自己抵押是0，赎回添自己账户可以赎回，72小时后到账，如果到不了需执行退款action,赎回给别人租借的需在租借列表执行赎回


```
       switch (params.actionState) {
                            case EOSUtils.ACTION_VOTE:
                                EOSVoteProducer voteProducer = new EOSVoteProducer(params.from, "", params.producersList);
                                actionList = ChainManager.getInstance().createVote(voteProducer,permissionLevelList);
                                break;
                            case EOSUtils.ACTION_UNDELEGATEBW:
                                EOSUndelegatebw Undelegatebw = new EOSUndelegatebw(params.from, params.to, new TypeAsset(params.net_quantity, token_name), new TypeAsset(params.cpu_quantity, token_name));
                                actionList = ChainManager.getInstance().createUndelegatebw(Undelegatebw,permissionLevelList);
                                break;
                            case EOSUtils.ACTION_UNDELEGATEBW_RENT:
                                EOSUndelegatebw Undelegatebw_rent = new EOSUndelegatebw(params.from, params.to, new TypeAsset(params.net_quantity, token_name), new TypeAsset(params.cpu_quantity, token_name));
                                actionList = ChainManager.getInstance().createUndelegatebw(Undelegatebw_rent,permissionLevelList);
                                break;
                            case EOSUtils.ACTION_UNDELEGATEBW_RENT_LIST:
                                List<EosRentList.Rows> eosRentList = params.rentList;
                                List<EOSUndelegatebw> Undelegatebw_rentList = new ArrayList<>();
                                for(EosRentList.Rows eosRent : eosRentList){
                                    String from = EosAccountDBHelper.getInstance().getMainEosAccount();
                                    String to = eosRent.getTo();
                                    String cpuValue = formAmount(eosRent.getCpu_weight());
                                    String netValue = formAmount(eosRent.getNet_weight());
                                    if(TextUtils.isEmpty(from) ||TextUtils.isEmpty(to) ){
                                        break;
                                    }
                                    EOSUndelegatebw UndelegatebwBean = new EOSUndelegatebw(from, to, new TypeAsset(netValue, token_name), new TypeAsset(cpuValue, token_name));
                                    Undelegatebw_rentList.add(UndelegatebwBean);

                                }
                                if(Undelegatebw_rentList == null){
                                    break;
                                }
                                actionList = ChainManager.getInstance().createUndelegatebwList(Undelegatebw_rentList,permissionLevelList);
                                break;
                            case EOSUtils.ACTION_DELEGATEBW:
                                //  boolean transfer  //给自己必须0   //给别人抵押可 0 和1 0代表租界 1抵押 代表
                                if (params.from.equals(params.to)) {
                                    params.transfer = false;
                                }
                                EOSDelegatebw delegatebw = new EOSDelegatebw(params.from, params.to, new TypeAsset(params.net_quantity, token_name), new TypeAsset(params.cpu_quantity, token_name), params.transfer);
                                actionList = ChainManager.getInstance().createDelegatebw(delegatebw,permissionLevelList);
                                break;

                            case EOSUtils.ACTION_BUY_RAM:
                                EOSBuyram buyram = new EOSBuyram(params.from,params.to ,new TypeAsset(params.quantStr, token_name));
                               actionList = ChainManager.getInstance().createBuyram(buyram,permissionLevelList);
                                break;

                            case EOSUtils.ACTION_REFOUND:
                                EOSRefund refund = new EOSRefund(params.from);
                                actionList = ChainManager.getInstance().createEOSRefund(refund,permissionLevelList);
                                break;
                            case EOSUtils.ACTION_SELL_RAM:
                                EOSellRam sellRam = new EOSellRam(params.from,params.quant);
                                actionList = ChainManager.getInstance().createSellRam(sellRam,permissionLevelList);
                                break;
    

```

#### DAPP

 AToken 以太坊DAPP遵循meadmask协议,EOS DAPP是Scatter协议，都需要通过注入js,来拦截DAPP请求
 
 
 EOS DAPP  两个重要回调登录账户和授权签名交易参数，传参给DAPP账户名，公钥，权限类型，公钥权限类型需通过，账户查询本地数据库来判端，授权签名，
 DAPP会返回交易参数，返回多个action，这时候需要把这些信息通过EOS主网节点的接口来解码，action不止一个所以要循环查找，解码后要给用户展示要签名的信息有那些，如action是转账，需要给用户展示转账地址，金额，备注等  
 
 
 
 #### 两个重要回调登录账户和授权签名交易参数
 ```
   if (methodName.equals("getEosAccount")) {    登录账户
            DebugLog.d(TAG, "getEosAccount ");
            responseBean.setCode(0);
            responseBean.setMessage("success");
            HashMap<String, String> hashMap = new HashMap();
            hashMap.put("account", mAccount);                               //登录账户
            hashMap.put("publicKey", EOSUtils.getPubkeyByType(mAccount));  //公钥
            hashMap.put("authority", EOSUtils.getPermissionsType(mAccount)); // 权限类型
            responseBean.setData(hashMap);
            JsCallback(serialNumber, new Gson().toJson(responseBean));
        }
	
	
if (methodName.equals("requestSignature")) {                          //客服端授权签名后，DAPP来执行合约交易
            DebugLog.d(TAG, "requestSignature " + params);
            if(isloading){  //如果在处理数据时，不在接收DAPP的返回
                return;
            }

            if (params.equals("{}") || TextUtils.isEmpty(params)) {
                parmsNullTransaction();
                return;
            }

            try {
                if (actionList != null) {
                    actionList.clear();
                }
                scatterSignBean = new Gson().fromJson(params, ScatterSignBean.class);
                isloading = true;
                for (int i = 0; i < scatterSignBean.getTransaction().getActions().size(); i++) {
                    enCodeData(i);//解码交易参数
                }
	
	
```
 
#### 执行签名前DAPP返回的data

```
{
	"transaction": {
		"expiration": "2018-12-12T09:43:30",
		"ref_block_num": 31782,
		"ref_block_prefix": 3862596156,
		"max_net_usage_words": 0,
		"max_cpu_usage_ms": 0,
		"delay_sec": 0,
		"context_free_actions": [],
		"actions": [{
			"account": "eosio.token",
			"name": "transfer",
			"authorization": [{
				"actor": "gy4tsmjvhege",
				"permission": "active"
			}],
			"data": "a0986afb499c8967309d4c462197b23a102700000000000004454f530000000040616374696f6e3a6265742c736565643a544b72353665724345506f4d5156357442372c726f6c6c556e6465723a35302c7265663a637075656d657267656e6379"
		}],
		"transaction_extensions": []
	},
	"buf": {
		"type": "Buffer",
		"data": [172, 163, 118, 242, 6, 184, 252, 37, 166, 237, 68, 219, 220, 102, 84, 124, 54, 198, 195, 62, 58, 17, 159, 251, 234, 239, 148, 54, 66, 240, 233, 6, 66, 216, 16, 92, 38, 124, 60, 138, 58, 230, 0, 0, 0, 0, 1, 0, 166, 130, 52, 3, 234, 48, 85, 0, 0, 0, 87, 45, 60, 205, 205, 1, 160, 152, 106, 251, 73, 156, 137, 103, 0, 0, 0, 0, 168, 237, 50, 50, 97, 160, 152, 106, 251, 73, 156, 137, 103, 48, 157, 76, 70, 33, 151, 178, 58, 16, 39, 0, 0, 0, 0, 0, 0, 4, 69, 79, 83, 0, 0, 0, 0, 64, 97, 99, 116, 105, 111, 110, 58, 98, 101, 116, 44, 115, 101, 101, 100, 58, 84, 75, 114, 53, 54, 101, 114, 67, 69, 80, 111, 77, 81, 86, 53, 116, 66, 55, 44, 114, 111, 108, 108, 85, 110, 100, 101, 114, 58, 53, 48, 44, 114, 101, 102, 58, 99, 112, 117, 101, 109, 101, 114, 103, 101, 110, 99, 121, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
	}
}
```
#### 解码后的信息

```
"args": {
    		"to": "betdiceadmin",
    		 "memo": "action:bet,
    		 rollUnder:50,
    		 ref:cpuemergency",
    		"from": "gy4tsmjvhege",
    		"quantity": "1.0000 EOS"}
      }
```





1. 沟通和交流


    QQ 交流群1
    
        

  
