

# Windchill开发接口

## 1. 部件



### 1.1 部件本身相关

#### 创建部件

```java
/**
 * 创建部件
 * @param number 编号, =null时自动编号
 * @param name 名称
 * @param subType 软类型
 * @param endItem 是否成品
 * @param unit 默认单位, =null时为“根据需要”
 * @param state 状态, =null时为生命周期初始化状态
 * @param view 视图, =null时为“设计视图”
 * @param folder 文件夹, =null时为容器根路径
 * @param container 容器, 与folder参数必填其一
 * @return 部件实例
 * @throws Exception
 */
public static WTPart createWTPart(String number, String name, String subType, boolean endItem, QuantityUnit unit, State state, View view, Folder folder, WTContainer container) throws Exception{
   WTPart part = WTPart.newWTPart();
   //number
   if(number != null)
      part.setNumber(number);
   //name
   part.setName(name);
   //subType 设置软类型
   if(subType != null){
      TypeDefinitionReference tdf = TypedUtilityServiceHelper.service.getTypeDefinitionReference(subType);
      part.setTypeDefinitionReference(tdf);
   }
   //endItem 是否为成品
   part.setEndItem(endItem);
   //unit 设置默认单位
   part.setDefaultUnit(unit == null ? QuantityUnit.AS_NEEDED : unit);
   //source 数据源
   part.setSource(Source.MAKE);
   //view
   if(view == null)
      view = ViewHelper.service.getView("Design");
   part.setView(ViewReference.newViewReference(view));
   //folder or contanier
   if(folder != null){
      FolderHelper.assignLocation((FolderEntry)part, folder);
   }else{
      part.setContainer(container);
   }
   part = (WTPart) PersistenceHelper.manager.save(part);
   //state
   if(state != null)
      part = (WTPart)LifeCycleHelper.service.setLifeCycleState(part, state);
   return part;
}
```

#### 父子部件建立LINK

```java
/**
 * 创建部件使用关系
 * @param part 父部件
 * @param master 子部件主数据
 * @param amount 数量
 * @param unit 单位, =null时为子部件默认单位
 * @param linenumber 行号, <0时不设置
 * @param findnumber 检索号, =null时不设置
 * @return 部件使用关系实例
 * @throws Exception
 */
public static WTPartUsageLink createWTPartUsageLink(WTPart part, WTPartMaster master, double amount, QuantityUnit unit, long linenumber, String findnumber) throws Exception{
   //uses and used
   WTPartUsageLink link = WTPartUsageLink.newWTPartUsageLink(part, master);
   //amount and unit
   if(unit == null)
      unit = master.getDefaultUnit();
   link.setQuantity(Quantity.newQuantity(amount, unit));
   //linenumber
   if(linenumber >=0)
      link.setLineNumber(LineNumber.newLineNumber(linenumber));
   //findnumber
   if(findnumber != null)
      link.setFindNumber(findnumber);
   
   PersistenceServerHelper.manager.insert(link);
   return link;
}
```

### 

#### 获取部件的最新版本

```java
//这个方法获取的视图是未知的
public static WTPart getLatestPart(WTPartMaster master) throws WTException{
   return (WTPart) QsVersionUtil.getLatest(master);
}
```

#### 获取指定视图的最新部件

```java
public static WTPart getPartWithView(WTPartMaster master, String viewName) throws WTException{
   QueryResult qr = VersionControlHelper.service.allVersionsOf(master);
   while(qr.hasMoreElements()){
      WTPart part = (WTPart) qr.nextElement();
      if(viewName.equals(part.getViewName()))
         return part;
   }
   return null;
}
```

####  部件的视图

```java
//通过视图的字符串获取视图对象
ViewHelper.service.getView("Design");
//获取视图对象的 viewReference 类型,用于直接给part.setView使用
ViewReference.newViewReference(view)
```

#### 获取子部件

```java
public void getChildrenPart(WTPart part){
  QueryResult qr = WTPartHelper.service.getUsesWTPartMasters(part);
		while (qr.hasMoreElements()) {
      //获取父部件与子部件的link关系
			WTPartUsageLink link = (WTPartUsageLink) qr.nextElement();
      //获取子部件的对象
			WTPart subPart = PartUtil.getLatestWTPart(link.getUses());

		}
  
}
```

#### 获取父部件

~~~java
//注意，会得到多个小版本，多个视图的部件
QueryResult qr = WTPartHelper.service.getUsedByWTParts(pPart.getMaster()); 
~~~



#### 删除link

~~~java
//找到link之后删除就好了 最好直接传 link的Oid
PersistenceHelper.manager.delete(link);
//不需要检入检出
PersistenceServerHelper.manager.remove(link);
~~~

#### 关于OID的VR与OR

VR 可以通过这个获取对象的最新小版本,而OR获取到的是对象的某个单一的版本

### 1.2 取部件的相关文档

#### 获取说明方文档

```java
QueryResult qr = WTPartHelper.service.getDescribedByDocuments(part);
```

#### 获取参考文档

~~~java
QueryResult qr = WTPartHelper.service.getReferencesWTDocumentMasters(part);
~~~

#### 获取三维图纸

```java
public static EPMDocument getAssociateEPM(WTPart part) throws WTException{
   QueryResult qr = PersistenceHelper.manager.navigate(part, EPMBuildRule.BUILD_SOURCE_ROLE, EPMBuildRule.class, false);
   while (qr.hasMoreElements()) {
      EPMBuildRule rule = (EPMBuildRule) qr.nextElement();
      if (7 == rule.getBuildType()) {
         EPMDocument epm = (EPMDocument) rule.getBuildSource();
         epm = getLatetNotInWorkSpace((EPMDocumentMaster) epm.getMaster());
         return epm;
      }
   }
   
   return null;
}
```

#### 获取二维图纸

```java
public static EPMDocument getDrawing(WTPart part) throws WTException{
   EPMDocument epm = getAssociateEPM(part);
   if(epm == null)
      return null;
   if(QsEPMUtil.isDwg(epm))
      return epm;
   return get2DDRW(epm);
}

public static EPMDocument getAssociateEPM(WTPart part) throws WTException{
		QueryResult qr = PersistenceHelper.manager.navigate(part, EPMBuildRule.BUILD_SOURCE_ROLE, EPMBuildRule.class, false);
		while (qr.hasMoreElements()) {
			EPMBuildRule rule = (EPMBuildRule) qr.nextElement();
			if (7 == rule.getBuildType()) {
				EPMDocument epm = (EPMDocument) rule.getBuildSource();
				epm = getLatetNotInWorkSpace((EPMDocumentMaster) epm.getMaster());
				return epm;
			}
		}
		
		return null;
}


	public static boolean isDwg(EPMDocument epm) {
		return epm != null && epm.getCADName().toUpperCase().endsWith(".DWG");
	}



public static EPMDocument get2DDRW(EPMDocument doc) throws WTException{
		String cadName = doc.getCADName().trim().toUpperCase();
		if(cadName.endsWith(".DRW"))
			return doc;

		EPMDocument epm = null;
		//先在三维模型的参考方文档中找同名的DRW
		cadName = cadName.substring(0, doc.getCADName().lastIndexOf("."));
		String strRegex = cadName + "\\.DRW";
		
		QueryResult qr = PersistenceHelper.navigate(doc.getMaster(), EPMReferenceLink.ROLE_AOBJECT_ROLE, EPMReferenceLink.class, true);
		qr = new LatestConfigSpec().process(qr);
		while(qr.hasMoreElements()){
			Object o = qr.nextElement();
			if(o instanceof EPMDocument){
				EPMDocument drw = (EPMDocument)o;
				if(Pattern.matches(strRegex, drw.getCADName().trim().toUpperCase())){
					epm = drw;
					break;
				}
			}
		}
		
		//若没找到,再使用查询找同名的DRW
		if(epm == null)
			epm = EPMUtil.findEPMbySuffix(doc, "DRW");
		
		return epm; 
	}
```

### 1.3 替换件相关

#### 获取局部替换件(link)

```java
public static WTCollection getSubstitutes(WTPartUsageLink link) throws WTException {
    WTArrayList arrayList = new WTArrayList();
    if (link != null) {
        WTCollection substituteLinks = WTPartHelper.service.getSubstituteLinks(link);
        arrayList.addAll(substituteLinks);
    }
    return arrayList;
}
```

#### 创建替换件之间的关系

```java
 public static void copySubPartByLink(WTPartUsageLink eLink, WTPartUsageLink mLink) throws WTException {
        WTPartSubstituteLink substituteLink = WTPartSubstituteLink.newWTPartSubstituteLink(WTPartUsageLink wtPartUsageLink, WTPartMaster wtPartMaster);
                substituteLink.setQuantity(SubstituteQuantity.newSubstituteQuantity(oldSubstitute.getQuantity().getAmount(), oldSubstitute.getSubstitutes().getDefaultUnit()));
                PersistenceServerHelper.manager.insert(substituteLink);
    }
```

#### 获取全局替换件

~~~java
    /**
     * 获取全局替换件的部件编号
     *
     * @param master
     * @return
     * @throws WTException
     */
    public static Set<String> getGlobalSubstitutes(WTPartMaster master) throws WTException {
        Set<String> set = new HashSet<>();
        if (master != null) {
          	//重点
            WTCollection substitutes = WTPartHelper.service.getAlternateLinks(master);
            Iterator iterator = substitutes.persistableIterator();
            while (iterator.hasNext()) {
                Object object = iterator.next();
                if (object instanceof WTPartAlternateLink) {
                    WTPartAlternateLink link = (WTPartAlternateLink) object;
                    set.add(link.getAlternates().getNumber());
                }
            }
        }
        return set;
    }

~~~

#### 创建全局替换件(特殊)

全局替换件不用单独处理,因为全局替换件跟着部件而不是跟着link

### 1.4 部件与文档之间建立关系

### 拥有的状态

INWORK 正在工作 (部件的状态) 

RELEASED 已发布 (已发布)

CANCELLED 已取消 (已取消)

DESIGNING 设计中(适用于文档)





## 2. 文档

### 2.1 文档本身

#### 创建文档

```java
			  WTDocument doc = WTDocument.newWTDocument();
        String docNumber = SunStrUtil.nextSequence(WTDocument.class);
        doc.setName(docName);
        doc.setNumber(docNumber);
        doc.setDescription(mdmDesc);
        doc.setTypeDefinitionReference(TypedUtility.getTypeDefinitionReference(typeDefStr));
        FolderHelper.assignFolder(doc, getDocumentLocation());
        doc = (WTDocument) PersistenceHelper.manager.save(doc);
```

#### 创建文档与文档之间的关系

```java
WTDocument parentDoc = QsDocUtil.findLatestDoc("0000000161");
WTDocument childDoc = QsDocUtil.findLatestDoc("0000000164");
Mastered master = childDoc.getMaster();
WTDocumentUsageLink link = WTDocumentUsageLink.newWTDocumentUsageLink(parentDoc, (WTDocumentMaster) master);
PersistenceHelper.manager.save(link);
```

#### 查找文档的子文档

```java
QueryResult qr = WTDocumentHelper.service.getUsesWTDocumentUsageLinks(doc);
while(qr.hasMoreElements()){
    WTDocumentUsageLink link =(WTDocumentUsageLink)qr.nextElement();
    WTDocumentMaster childDocMaster = (WTDocumentMaster) link.getRoleBObject();
    WTDocument childDoc = QsDocUtil.getLatestDocument(childDocMaster);
}
```

#### 删除文档

```
		//获取文档对象直接删除
		PersistenceHelper.manager.delete(docs);
```

#### 创建部件与文档的关系

##### 说明方文档

~~~java
	WTPartDescribeLink partDocLink = WTPartDescribeLink.newWTPartDescribeLink(part, doc);
	PersistenceServerHelper.manager.insert(partDocLink);
~~~

#### 获取主要内容

~~~java
	//获取主要内容对象
	ContentItem contentitem = ContentHelper.service.getPrimary(doc)
  //将主要内容转为流
	ApplicationData applicationData = (ApplicationData) contentitem;	
	return applicationData == null ? null : ContentServerHelper.service.findLocalContentStream(applicationData);
~~~



#### 为文档上传主要内容

```java
//更新文档的Primary的内容
ApplicationData applicationdata = ApplicationData.newApplicationData(doc);
applicationdata.setRole(ContentRoleType.PRIMARY);
applicationdata = ContentServerHelper.service.updateContent(doc, applicationdata, appPath);
doc = (WTDocument) PersistenceHelper.manager.refresh(doc);// 刷新文档
```

注意里面放的都是文件的路径

#### 上传附件

```java
ApplicationData ad = null;
for (String secondaryFilePath : list) {
    ad = ApplicationData.newApplicationData(doc);
    ad.setRole(ContentRoleType.SECONDARY);
    ContentServerHelper.service.updateContent(doc, ad, secondaryFilePath);
}
```

### 2.2 获取文档相关对象

#### 获取参考文档

```jade
public static List<WTDocument> getReferencesByWTDocument(WTDocument doc) throws WTException {
    List<WTDocument> doclist = new ArrayList<WTDocument>();
    QueryResult qr = WTDocumentHelper.service.getDependsOnWTDocuments(doc);
    while (qr.hasMoreElements()) {
        WTDocument tdoc = (WTDocument) qr.nextElement();
        doclist.add(tdoc);
    }
    return doclist;
}
```

#### 获取参考方部件

```java
/*
 * 获取参考文档关联的部件
 */
public static Set<WTPartMaster> getReferencePart(WTDocument doc) throws WTException{
   Set<WTPartMaster> set = new HashSet<WTPartMaster>();
   QueryResult qrE = PersistenceHelper.manager.navigate(doc.getMaster(), "referencedBy", WTPartReferenceLink.class, true);
   while(qrE.hasMoreElements()){
      WTPart part = (WTPart)qrE.nextElement();
      set.add(part.getMaster());
   }
   return set;
}
```



## 3. EPMDocument

### 创建EBOM文档

```java
/**
 * @Title: createEPMDocument
 * @Description: 根据传递的信息,创建文档
 * @param container
 *            上下文
 * @param folder
 *            文件夹
 * @param docName
 *            文档名称
 * @param number
 *            文档编号
 * @param primaryFileName
 *            做为主内容的文件路径
 * @return void
 * @throws Exception
 */
public static EPMDocument createEPMDoc(String number,WTContainer container, Folder folder,String docName,String primaryFileName) throws Exception{
   EPMDocument doc = null;
   try{
      doc = EPMDocument.newEPMDocument();
      // 设置文档名称
         doc.setName(docName);
      // 设置文档编号
         doc.setNumber(number);
      // 设置文档类型
         wt.type.TypeDefinitionReference tdr = TypedUtility.getTypeDefinitionReference("wt.epm.EPMDocument");
         doc.setTypeDefinitionReference(tdr);
      // 设置上下文
         doc.setContainer(container);
      // 设置文件夹
         FolderHelper.assignLocation((FolderEntry)doc, folder);
      // 生成文档
         doc = (EPMDocument)PersistenceHelper.manager.store(doc);
         
         if(StringUtils.isNotBlank(primaryFileName)){
         // 上传主内容
            ApplicationData primaryContent = (ApplicationData) ContentHelper.service.getPrimaryContent(ObjectReference.newObjectReference(doc));
            if(primaryContent == null){
               primaryContent = ApplicationData.newApplicationData(doc);
               primaryContent.setRole(ContentRoleType.PRIMARY);
            }
            primaryContent.setUploadedFromPath(primaryFileName);
            primaryContent = ContentServerHelper.service.updateContent(doc, primaryContent, primaryFileName);
         }
         
      System.out.println("文档创建成功：文档编号[" + doc.getNumber() + "],名称[" + doc.getName() + "]");
         
         return doc;
   } catch(Exception e){
      e.printStackTrace();
      return null;
   } 

}
```

### 获取EPM相关联的部件

```java
    public static WTPart getAssociatePart(EPMDocument doc) throws WTException{      
      QueryResult qr = PersistenceHelper.manager.navigate(doc, EPMBuildRule.BUILD_TARGET_ROLE, EPMBuildRule.class, true);
      qr = new LatestConfigSpec().process(qr);
      
      if(qr.size() == 1){
         return (WTPart)VersionControlHelper.getLatestIteration((Iterated)qr.nextElement());
      }else if(qr.size() > 1){
         Mastered master = ((WTPart)qr.nextElement()).getMaster();
         return (WTPart)VersionControlHelper.service.allVersionsOf(master).nextElement();
      }else{
         return  null;
      }
   }  
```



### 获取二维图纸对象 不熟

```java
public static EPMDocument get2DDRW(EPMDocument doc) throws WTException{
   String cadName = doc.getCADName().trim().toUpperCase();
   if(cadName.endsWith(".DRW"))
      return doc;

   EPMDocument epm = null;
   // 先在三维模型的参考方文档中找同名的DRW
   cadName = cadName.substring(0, doc.getCADName().lastIndexOf("."));
   String strRegex = cadName + "\\.DRW";
   
   QueryResult qr = PersistenceHelper.navigate(doc.getMaster(), EPMReferenceLink.ROLE_AOBJECT_ROLE, EPMReferenceLink.class, true);
   qr = new LatestConfigSpec().process(qr);
   while(qr.hasMoreElements()){
      Object o = qr.nextElement();
      if(o instanceof EPMDocument){
         EPMDocument drw = (EPMDocument)o;
         if(Pattern.matches(strRegex, drw.getCADName().trim().toUpperCase())){
            epm = drw;
            break;
         }
      }
   }
   return epm; 
}
```

## 4. 升级请求

### 创建升级请求

```java
/**
 * 创建升级请求
 *
 * @param name
 * @param wtSet
 * @param wfpName
 * @param product
 * @param oid     顶层oid
 * @return
 * @throws Exception
 */
public PromotionNotice createNewPromotionNotice(String name, WTSet wtSet, String wfpName, WTContainer product, String oid, String parentOid) throws Exception {
    Transaction tran = null;
    PromotionNotice pn = PromotionNotice.newPromotionNotice(name);
    try {
        tran = new Transaction();
        tran.start();
        WTPart part = (WTPart) Util.getObjByOid(oid);
        pn.setContainer(product);
        // 获取软类型
        TypeInstanceIdentifier tii = getSoftType(PromotionNotice.class, product, "FBomRelease");
        //当前用户
        WTUser user = (WTUser) SessionHelper.getPrincipal();
        // 设置当前用户为创建用户
        SessionHelper.manager.setPrincipal(user.getName());
        if (tii != null) {
            // 实例化对象
            TypeInstance ti = TypeInstanceFactory.newTypeInstance(tii);
            pn = (PromotionNotice) ServerCommandDelegateUtility.translate(ti, null);
            pn.setName(name);
            JSONObject obj = new JSONObject();
            obj.put("oid", oid);
            //如果两个部件id 相同 则不写入parentOid
            if (oid.equalsIgnoreCase(parentOid)) {
                obj.put("parentOid", "");
            } else {
                obj.put("parentOid", parentOid);
            }
            obj.put("number", part.getNumber());
            obj.put("viewName", part.getViewName());
            pn.setDescription(obj.toJSONString());
            Folder folder = (Folder) QsFolderUtil.getFolder(product, "/Default/");
            pn.setFolderingInfo(FolderingInfo.newFolderingInfo(folder));
            FolderHelper.assignFolder(pn, folder);
            //新建基线,并设置到升级请求中(重要的设置)
            MaturityBaseline maturityBaseline = MaturityBaseline.newMaturityBaseline();
            maturityBaseline.setContainerReference(WTContainerRef.newWTContainerRef(product));
            maturityBaseline = (MaturityBaseline) PersistenceHelper.manager.save(maturityBaseline);
            pn.setConfiguration(maturityBaseline);
            MaturityHelper.service.savePromotionNotice(pn);
            MaturityHelper.service.savePromotionTargets(pn, wtSet);
            //添加升级请求的对象到基线
            BaselineHelper.service.addToBaseline(wtSet, maturityBaseline);
            // 获取工作流程模板
            WfProcessTemplate wfpt = getWFPTByName(wfpName);
            System.out.println(wfpt);
            System.out.println(pn);
            if (wfpt != null && pn != null) {
                // 启动升级进程(通过工作流模板)
                MaturityHelper.service.startPromotionProcess(pn, wfpt);
                System.out.println("升级请求创建完成====> " + pn.getDisplayIdentifier());
            } else {
                throw new Exception("工作流模板为空,或者升级请求为空!");
            }
        }
        tran.commit();
        tran = null;
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        if (tran != null) {
            tran.rollback();
        }
    }
    return pn;
}
```



### 获取升级对象

~~~java

QueryResult qr = MaturityHelper.service.getPromotionTargets(pn);
while (qr.hasMoreElements()) {
        Object obj = qr.nextElement();
  
}

~~~



### 删除升级对象中的内容

```java
/**
 * 校验升级请求中是否包含技术体系文档
 * 如果有则删除该升级对象
 *
 * @param pn
 */
public static void checkPnHasTIDocAndRemoveIt(PromotionNotice pn) throws WTException {
    QueryResult qr = MaturityHelper.service.getPromotionTargets(pn);
    WTDocument document = null;
    while (qr.hasMoreElements()) {
        Object obj = qr.nextElement();
        if (obj instanceof WTDocument) {
            WTDocument doc = (WTDocument) obj;
            String type = WTUtil.getSoftType(doc);
            //文档需要是产品技术文档
            if ("TIDoc".equals(type)) {
                document = doc;
                break;
            }
        }
    }
    if (document != null) {
        MaturityBaseline baseline = pn.getConfiguration();
        WTSet set = new WTHashSet();
        set.add(document);
        MaturityHelper.service.deletePromotionTargets(pn, set);
        BaselineHelper.service.removeFromBaseline(set, baseline);
    }
}
```





## 5. ECN 更改通告

## 6. 通用API

### 1. 通过OID获取对象

~~~java
public static WTObject getObjByOid(String oid) throws WTException{
      ReferenceFactory referencefactory = new ReferenceFactory();
	    WTObject obj = (WTObject) referencefactory.getReference(oid).getObject();
			return obj;
}
~~~

### 2. 通过对象获取OID

~~~java
public static String getOid(WTObject obj){
	    	String oid ="";
	    	wt.fc.ReferenceFactory rf = new ReferenceFactory();
	    	try {
					oid = rf.getReferenceString(obj);
				} catch (WTException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
				}
	    		return oid;
}
~~~

### 3. 获取对象的最后一个版本(OID)

```java
public static WTObject getLatestObjByOid(String oid) throws WTException{
   ReferenceFactory referencefactory = new ReferenceFactory();
    WTObject obj = (WTObject) referencefactory.getReference(oid).getObject();
    obj = (WTObject)VersionControlHelper.service.getLatestIteration((Iterated)obj,false);
   return obj;
}
```

### 4. 通过master对象获取最新版本

```java
public static WTDocument getLatestDocument(WTDocumentMaster master)
      throws PersistenceException, WTException {
   QueryResult qr = VersionControlHelper.service.allVersionsOf(master);
   return (WTDocument) qr.nextElement();
}
```

### 5. 获取对象的软类型

```java
public static String getSoftType(Typed typed) throws WTException {
    try {
        String typeStr = ClientTypedUtility.getExternalTypeIdentifier(typed);
        return typeStr.substring(typeStr.lastIndexOf(".") + 1);
    } catch (Exception e) {
        throw new WTException(e);
    }
}
```

## 其他

### 获取序列

~~~java
//获取对象的编号序列 不是id 是编号 ,也许还可以传 WTPart.class
	//获取文档的序列 还有 	WTPARTID_SEQ T_TASK_ID_SEQ MPMPROCESSPLANID_SEQ
	String seq = PersistenceHelper.manager.getNextSequence("WTDOCUMENTID_SEQ");

~~~

### 获取IBA软属性的值

```java
public static String getIBAValue(Object obj, String ibaName)
      throws Exception {
   if (!RemoteMethodServer.ServerFlag) {
      return (String) RemoteMethodServer.getDefault().invoke(
            "getIBAValue", IBAUtils.class.getName(), null,
            new Class[] { Object.class, String.class },
            new Object[] { obj, ibaName });

   }
   String ibaValue = "";
   LWCNormalizedObject obj0 = new LWCNormalizedObject((Persistable) obj,
         null, Locale.US, new UpdateOperationIdentifier());
   obj0.load(ibaName);
   Object value = obj0.get(ibaName);
   
   if(value != null){
      if(value instanceof Object[]){
         Object[] list = (Object[]) value;
         for(int i = 0; i < list.length; i++){
            String val = list[i].toString();
            ibaValue += val + ",";
         }
         ibaValue = ibaValue.substring(0, ibaValue.length() - 1);
      }else{
         ibaValue = (String) value;
      }
   }
   return ibaValue;
}
```

## 7. 高级查询

### 7.1 查询部件

```java
/***
 * 检索物料
 * @param name 精确名称
 * @param likeName 模糊名称
 * @param number 精确编号
 * @param likeNumber 模糊编号
 * @param view 视图
 * 备注：精确与模糊字段同时存在时仅取精确
 * @return
 * @throws Exception
 */
private static QuerySpec searchPart(String name,String likeName,String number,String likeNumber,String view,String containerName,String createName) throws Exception{
   QuerySpec qs = new QuerySpec(); 
   int partIndex = qs.appendClassList(WTPart.class, true);
   qs.appendWhere(QsQueryUtil.WHERE_TRUE);
   //名称
   if(StringUtils.isNotBlank(name)){
      qs.appendAnd();
      qs.appendWhere(new SearchCondition(WTPart.class, WTPart.NAME, SearchCondition.EQUAL, name), 0);
   }else if(StringUtils.isNotBlank(likeName)){
      qs.appendAnd();
      qs.appendWhere(new SearchCondition(WTPart.class, WTPart.NAME, SearchCondition.LIKE, QsQueryUtil.buildLikeQueryStr(likeName)), 0);
   }
   //编号
   if(StringUtils.isNotBlank(number)){
      qs.appendAnd();
      qs.appendWhere(new SearchCondition(WTPart.class, WTPart.NUMBER, SearchCondition.EQUAL, number), 0);
   }else if(StringUtils.isNotBlank(likeNumber)){
      qs.appendAnd();
      qs.appendWhere(new SearchCondition(WTPart.class, WTPart.NUMBER, SearchCondition.LIKE, QsQueryUtil.buildLikeQueryStr(likeNumber)), 0);
   }
   //上下文
   if(StringUtils.isNotBlank(containerName)){
      int ctIndex = qs.appendClassList(WTContainer.class, false);
      qs.appendAnd();
      qs.appendWhere(new SearchCondition(WTPart.class, "containerReference.key.id", WTContainer.class, WTAttributeNameIfc.ID_NAME) , 0, ctIndex);
      qs.appendAnd();
      qs.appendWhere(new SearchCondition(WTContainer.class, WTContainer.NAME, SearchCondition.EQUAL, containerName), ctIndex);
   }
   //创建者
   if(StringUtils.isNotBlank(createName)){
      int usrIndex = qs.appendClassList(WTUser.class, false);
      qs.appendAnd();
      qs.appendWhere(new SearchCondition(WTPart.class, "iterationInfo.creator.key.id", WTUser.class, WTAttributeNameIfc.ID_NAME) , 0, usrIndex);
      qs.appendAnd();
      qs.appendWhere(new SearchCondition(WTUser.class, WTUser.NAME, SearchCondition.EQUAL, createName), usrIndex);
   }
   if(StringUtils.isNotBlank(view)){
      qs.appendAnd();
      qs.appendWhere(new SearchCondition(WTPart.class, WTPart.VIEW + "." + WTAttributeNameIfc.REF_OBJECT_ID,SearchCondition.EQUAL, ViewHelper.service.getView(view).getPersistInfo().getObjectIdentifier().getId()), 0);
   }
   return qs;
}
```

### 7.2 查询文档

```java
/**
     * 根据条件检索文档
     *
     * @param name          名称 优先精确查询
     * @param likeName      名称(模糊查询)
     * @param number        编号 优先精确查询
     * @param likeNumber    编号(模糊查询)
     * @param dcc           dcc编码
     * @param state         状态（英文）
     * @param containerName 容器名称
     * @param createName    创建人ID
     * @param folders       文件夹ID集合
     * @param versionType   版本类型
     * @param version       大版本
     * @param iteration     小版本
     * @param firstTime     开始时间
     * @param endTime       结束时间
     * @param docType       文档类型
     * @return
     * @throws Exception
     */
    private static QuerySpec searchDoc(String name, String likeName, String number, String likeNumber, String dcc, String likeDcc,
                                       String state, String containerName, String createName, String createFullName, long[] folders, String versionType, String version,
                                       String iteration, String firstTime, String endTime, String fileType, String docType) throws Exception {
        if(StringUtils.isBlank(name) && StringUtils.isBlank(likeName) && StringUtils.isBlank(number) &&
         StringUtils.isBlank(likeNumber) && StringUtils.isBlank(dcc) && StringUtils.isBlank(likeDcc) && 
         StringUtils.isBlank(state) && StringUtils.isBlank(containerName) && StringUtils.isBlank(createName) && 
         StringUtils.isBlank(createFullName) && StringUtils.isBlank(versionType) && StringUtils.isBlank(version) && 
         StringUtils.isBlank(iteration) && StringUtils.isBlank(firstTime) && StringUtils.isBlank(endTime) && 
         StringUtils.isBlank(fileType) && StringUtils.isBlank(docType) && folders == null) {
           number = "无";
        }
       QuerySpec qs = new QuerySpec();
        int docIndex = qs.appendClassList(WTDocument.class, true);
        qs.appendWhere(QsQueryUtil.WHERE_TRUE);
        //名称
        if (StringUtils.isNotBlank(name)) {
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(WTDocument.class, WTDocument.NAME, SearchCondition.EQUAL, name), 0);
        } else if (StringUtils.isNotBlank(likeName)) {
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(WTDocument.class, WTDocument.NAME, SearchCondition.LIKE, QsQueryUtil.buildLikeQueryStr(likeName)), 0);
        }
        //编号
        if (StringUtils.isNotBlank(number)) {
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(WTDocument.class, WTDocument.NUMBER, SearchCondition.EQUAL, number), 0);
        } else if (StringUtils.isNotBlank(likeNumber)) {
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(WTDocument.class, WTDocument.NUMBER, SearchCondition.LIKE, QsQueryUtil.buildLikeQueryStr(likeNumber)), 0);
        }
        //状态
        if (StringUtils.isNotBlank(state)) {
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(WTDocument.class, WTDocument.LIFE_CYCLE_STATE, SearchCondition.EQUAL, state), 0);
        }
        //文档类型
        if (StringUtils.isNotBlank(docType)) {
            qs.appendAnd();
            QsQueryUtil.appendTypeCondition(qs, docIndex, docType);
        }
        //DCC编码
        if (StringUtils.isNotBlank(dcc)) {
           qs.appendAnd();
            QsQueryUtil.appendIBACondition(qs, docIndex, "ZPLM_DOC_CHAR13", dcc);
        } else if (StringUtils.isNotBlank(likeDcc)) {
            qs.appendAnd();
            QsQueryUtil.appendIBACondition(qs, docIndex, "ZPLM_DOC_CHAR13", QsQueryUtil.buildLikeQueryStr(likeDcc), SearchCondition.LIKE, false);
        }
        //上下文
        if (StringUtils.isNotBlank(containerName)) {
            int ctIndex = qs.appendClassList(WTContainer.class, false);
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(WTDocument.class, "containerReference.key.id", WTContainer.class, WTAttributeNameIfc.ID_NAME), 0, ctIndex);
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(WTContainer.class, WTContainer.NAME, SearchCondition.EQUAL, containerName), ctIndex);
        }
        //创建者ID
        if (StringUtils.isNotBlank(createName)) {
            int usrIndex = qs.appendClassList(WTUser.class, false);
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(WTDocument.class, "iterationInfo.creator.key.id", WTUser.class, WTAttributeNameIfc.ID_NAME), 0, usrIndex);
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(WTUser.class, WTUser.NAME, SearchCondition.EQUAL, createName), usrIndex);

        }
        //创建者名称
        if (StringUtils.isNotBlank(createFullName)) {
            int usrIndex = qs.appendClassList(WTUser.class, false);
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(WTDocument.class, "iterationInfo.creator.key.id", WTUser.class, WTAttributeNameIfc.ID_NAME), 0, usrIndex);
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(WTUser.class, WTUser.FULL_NAME, SearchCondition.EQUAL, createFullName), usrIndex);
        }
        //文件夹
        if (folders != null && folders.length > 0) {
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(WTDocument.class, WTDocument.PARENT_FOLDER + ".key.id", folders, false), 0);
        }
        //所有大版本的最新小版本
        if ("newLargeVersion".equals(versionType)) {
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(WTDocument.class, WTDocument.LATEST_ITERATION, SearchCondition.IS_TRUE), docIndex);
        }
        //大版本
        if (StringUtils.isNotBlank(version)) {
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(WTDocument.class, "versionInfo.identifier.versionId", SearchCondition.EQUAL, version), 0);
        }
        //小版本
        if (StringUtils.isNotBlank(iteration)) {
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(WTDocument.class, "iterationInfo.identifier.iterationId", SearchCondition.EQUAL, iteration), 0);
        }
        //时间段
        if (StringUtils.isNotBlank(firstTime) && StringUtils.isNotBlank(endTime)) {
            Timestamp end = Timestamp.valueOf(endTime);
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(WTDocument.class, WTDocument.CREATE_TIMESTAMP, SearchCondition.LESS_THAN_OR_EQUAL, end), docIndex);
            qs.appendAnd();
            Timestamp start = Timestamp.valueOf(firstTime);
            qs.appendWhere(new SearchCondition(WTDocument.class, WTDocument.CREATE_TIMESTAMP, SearchCondition.GREATER_THAN_OR_EQUAL, start), docIndex);
        }
        //文件内容类型
        if (StringUtils.isNotBlank(fileType)) {
            int dfIndex = qs.appendClassList(DataFormat.class, false);
            int htcIndex = qs.appendClassList(HolderToContent.class, false);
            int appIndex = qs.appendClassList(ApplicationData.class, false);
//       qs.appendAnd();
//       qs.appendWhere(new SearchCondition(ApplicationData.class, ApplicationData.ROLE, SearchCondition.EQUAL, contentType), appIndex);
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(HolderToContent.class, "roleAObjectRef.key.id", WTDocument.class, WTAttributeNameIfc.ID_NAME), htcIndex, docIndex);
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(HolderToContent.class, "roleBObjectRef.key.id", ApplicationData.class, WTAttributeNameIfc.ID_NAME), htcIndex, appIndex);
            qs.appendAnd();
            FromClause fc = qs.getFromClause();
            qs.appendWhere(new SearchCondition(new TableColumn(fc.getAliasAt(appIndex), "ida3b4"), SearchCondition.EQUAL, new TableColumn(fc.getAliasAt(dfIndex), "IDA2A2")), appIndex, dfIndex);
            qs.appendAnd();
            qs.appendWhere(new SearchCondition(DataFormat.class, DataFormat.FORMAT_NAME, SearchCondition.EQUAL, fileType), dfIndex);
        }
        return qs;
    }
```

### 7.3 获取最新版本

```java
public static QueryResult findOnlyLatest(QuerySpec qs) throws WTException{
   QueryResult qr = PersistenceHelper.manager.find(qs);
   return processToLatest(qr, qs.getFromClause().getCount() > 1);
}

public static QueryResult processToLatest(QueryResult qr, boolean arrayElement) throws WTException{
   ObjectVector vector = new ObjectVector();
   while(qr.hasMoreElements()){
      if(arrayElement)
         vector.addElement(((Persistable[])qr.nextElement())[0]);
      else
         vector.addElement(qr.nextElement());
   }
   qr = new QueryResult(vector);
   qr = new LatestConfigSpec().process(qr);
   return qr;
}
```

## 8. 产品库相关

### 查询所有产品

```java
public static QueryResult getAllProductName() throws WTException {
    QuerySpec qs = new QuerySpec(PDMLinkProduct.class);
    return PersistenceHelper.manager.find(qs);
}
```



### 根据名称查询容器

```java
/**
 * 根据名称查找容器
 */
public static Set searchWTContainers(String containerName) {
   Set containers = new HashSet();
   if (containerName == null) {
      throw new IllegalArgumentException("The containerName is null!!!");
   }
   try {
      QuerySpec qs = new QuerySpec(WTContainer.class);
      qs.appendWhere(
            new SearchCondition(WTContainer.class, WTContainer.NAME,
                  SearchCondition.EQUAL, containerName.toUpperCase(),
                  false), new int[] { 0 });
      QueryResult qr = PersistenceHelper.manager.find((StatementSpec) qs);
      while (qr.hasMoreElements()) {
         containers.add(qr.nextElement());
      }
   } catch (WTException e) {
      e.printStackTrace();
   }
   return containers;
}
```

### 根据名称查询产品

```java
/**
 * 根据名称取得资源库
 */
public static PDMLinkProduct getPDMLinkProduct(String libraryName) {
    Set containers = PEditorUtil.searchWTContainers(libraryName);
    for (Iterator iterator = containers.iterator(); iterator.hasNext(); ) {
        Object object = iterator.next();
        if (object instanceof PDMLinkProduct) {
            return (PDMLinkProduct) object;
        }
    }
    return null;
}
```

### 根据软属性查询产品

```java
/**
 * 通过产品代号查找产品
 *
 * @param productNumber 产品代号
 * @return
 */
public static PDMLinkProduct getPDMLinkProductByProductNumber(String productNumber) {
    try {
        QuerySpec qs = new QuerySpec();
        int documentIndex = qs.appendClassList(PDMLinkProduct.class, true);
        QsQueryUtil.appendIBACondition(qs, documentIndex, "productNo", productNumber, SearchCondition.LIKE);
        QueryResult qr = PersistenceHelper.manager.find(qs);
        while (qr.hasMoreElements()) {
            Object[] obj1 = (Object[]) qr.nextElement();
            PDMLinkProduct product = (PDMLinkProduct) obj1[0];
            if (product != null) {
                return product;
            }
        }
        return null;
    } catch (Exception e) {
        e.printStackTrace();
        return null;
    }
}
```

### 9. 用户与组相关

#### 获取当前登陆人

~~~java
WTUser user = (WTUser) SessionHelper.getPrincipal();
~~~

#### 根据名称查找组

```java
public static WTGroup findGroup(String name) throws WTException{
   ExchangeContainer container = WTContainerHelper.service.getExchangeContainer();
   WTGroup group = OrganizationServicesHelper.manager.getGroup(name, container.getOrganization());//org
   if(group == null){
      OrganizationServicesHelper.manager.getGroups(name, container.getContextProvider());//site
   }
   
   if(group == null)
      throw new WTException("未找到名称为'" + name +"'的组.");
   return group;
}
```

判断是不是在组里main

```java
OrganizationServicesHelper.manager.isMember(group, user)
```

# 特殊修改

## 1. 修改windchill主页滚动的字幕

路径在

/home/ptc/Windchill_11.0/Windchill/codebase/netmarkets/jsp/util/begin_custom.jspf