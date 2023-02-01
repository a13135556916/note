# Windchill 其他功能

## Http接口

Windchill支持普通的Http方式调用接口，需要在接口的headers中传参 Authorization（键） 信息，即可实现调用

~~~
将用户名与密码进行base64 编码
System.out.println("Basic "+base.encode("wcadmin:pwst1234".getBytes()));
~~~

## 增加过滤按钮

### 新增类: AdministratorValidator

```java
/*
 *
 */
package ext.kaiyuan.customtaskmanage.validator;

import com.ptc.core.ui.validation.DefaultSimpleValidationFilter;
import com.ptc.core.ui.validation.UIValidationCriteria;
import com.ptc.core.ui.validation.UIValidationKey;
import com.ptc.core.ui.validation.UIValidationStatus;
import ext.kaiyuan.util.OrgUtil;
import wt.org.WTUser;
import wt.session.SessionHelper;

public class AdministratorValidator extends DefaultSimpleValidationFilter {

   @Override
   public UIValidationStatus preValidateAction(UIValidationKey uivalidationkey,
         UIValidationCriteria uivalidationcriteria) {
      UIValidationStatus uivalidationstatus = UIValidationStatus.HIDDEN;
      try {
         WTUser curUser = (WTUser) SessionHelper.manager.getPrincipal();
         boolean result = OrgUtil.isMemberOfGroup("KY_分类管理员组", curUser);
         if (result) {
            uivalidationstatus = UIValidationStatus.ENABLED;
         }
      } catch (Exception e) {
         e.printStackTrace();
      }
      return uivalidationstatus;
   }

}
```

### 修改KaiyuanValidator.xconf (可能是错的)

```xml
<Service context="default" name="com.ptc.core.ui.validation.SimpleValidationFilter">
       <Option serviceClass="ext.kaiyuan.customtaskmanage.validator.AdministratorValidator" selector="AdministratorValidator" requestor="null" />
</Service>
```

### 修改custom-actions.xml

```xml
<action name="knowledgeBase" resourceBundle="ext.kaiyuan.processdocmanage.resource.ProcessDocRB">
   <includeFilter name="AdministratorValidator" />
</action>

<action name="knowledgeBase" resourceBundle="ext.kaiyuan.processdocmanage.resource.ProcessDocRB">
   <includeFilter name="AdministratorValidator" />
</action>
```

### 运行命令 - 向windchill中注册这个filter

~~~shell
xconfmanager -s wt.services/svc/default/com.ptc.core.ui.validation.SimpleValidationFilter/ActionTest1DocStateFilter/null/0=ext.test.filter.ActionTest1DocStateFilter/duplicate -t codebase/service.properties -p
~~~

执行命令 xconfmanager -p

4. 重启

## 创建WebService服务(Java 原生方式)

### 1. 创建启动器、服务本身

~~~ java
package ext.pwst2.testWebService;

import javax.xml.ws.Endpoint;
import java.io.ByteArrayOutputStream;
import java.io.PrintStream;
import java.util.HashSet;
import java.util.Set;


public class TestAdapterManager {
	private static Set<Endpoint> Managed_Endpoints = new HashSet<Endpoint>();
	private static String HOST_IP = "172.16.10.107";//测试机
//	private static String HOST_IP = "10.254.100.17";//正式机
	static{
		try{
//			HOST_IP = InetAddress.getLocalHost().getHostAddress();
		}catch(Exception e){
			e.printStackTrace();
			System.out.println("-------AdapterManager 初始化失败---------");
		}
	}
	
	public static boolean isTestPlmServer(){
		return "172.16.10.107".equals(HOST_IP);
	}
	
	public static boolean publish() throws Exception{
		boolean success = true;
		String url = "http://" + HOST_IP + ":9999/" + "TestService";
		success = success && publish(url, TestWebService.class);
		System.out.println(">>>服务器IP地址: " + HOST_IP);
		System.out.println(">>>测试服务器[TestPlm]: " + isTestPlmServer());
		return success;
	}
	
	public static boolean publish(String address, Class serviceCls){
		if(address == null || serviceCls == null)
			return false;
		try{
			System.out.println("正在启动适配服务: [" + serviceCls.getName() + "]...");
			System.out.println("服务地址: " + address);
			Endpoint endpoint = Endpoint.publish(address, serviceCls.newInstance());
			if(endpoint != null && !Managed_Endpoints.contains(endpoint))
				Managed_Endpoints.add(endpoint);
			System.out.println("Webservice[" + serviceCls.getName() + "] 启动成功!");
			return true;
		}catch(Exception e){
			ByteArrayOutputStream bos = new ByteArrayOutputStream();
			e.printStackTrace(new PrintStream(bos));
			System.out.println("Webservice[" + serviceCls.getName() + "] 启动失败!\n错误信息:" + bos.toString());
			return false;
		}
	}
	
	public static void runStartBat() throws Exception{
		String folder = getRealPath(TestAdapterManager.class);
		String batPath = folder + "AdapterManager_Start.bat";
		String command = "cmd /c start " + batPath;
		Runtime.getRuntime().exec(command);
	}


	public static String getRealPath(Class class1){
		String folder = class1.getResource("").getPath();
		return folder.substring(1).replace('/', '\\');
	}

	public static void main(String[] args) throws Exception {
		if(!publish())
			Runtime.getRuntime().exit(1);
		System.out.println();
		System.out.println(">>> Running Adapter Count: " + Managed_Endpoints.size());
		System.out.println(">>> Windchill系统集成适配服务运行中，请勿在本shell窗口执行其他操作！");
	}
}

~~~

### 2. 创建WebService本身的服务

```java
package ext.pwst2.testWebService;

import wt.httpgw.GatewayAuthenticator;
import wt.method.RemoteMethodServer;

import javax.jws.WebMethod;
import javax.jws.WebService;

@WebService
public class TestWebService {
   @WebMethod
   public String findPartByNumber(String number) throws Exception{
      RemoteMethodServer rms = RemoteMethodServer.getDefault();
      GatewayAuthenticator auth = new GatewayAuthenticator();
      //使用该方式可以无需密码即可调用代码
      auth.setRemoteUser("wcadmin");
      rms.setAuthenticator(auth);
      return (String)rms.invoke("findPartByNumber", TestServiceHelper.class.getName(), null,
            new Class[]{String.class}, new Object[]{number});
   }

}
```

### 3.业务代码(最终被调用的)

```java
package ext.pwst2.testWebService;

import ext.pwst.util.QsPartUtil;
import wt.method.RemoteAccess;
import wt.part.WTPart;
import wt.util.WTException;

import java.text.SimpleDateFormat;
import java.util.Date;

public class TestServiceHelper implements RemoteAccess{
   private static final SimpleDateFormat sdf = new SimpleDateFormat("yyyyMMdd HH:mm:ss.SSS");
   
   /**
    * 根据编号查询部件信息
    * @param number 部件编号
    * @return
    * @throws Exception
    */
   public static String findPartByNumber(String number) throws WTException {
      Date now = new Date();
      now.setHours(now.getHours() + 8);
      System.out.println("@@##调用接口 findPartByNumber－－》" + sdf.format(now));
      WTPart part = QsPartUtil.findDesignPart(number);
      if (part==null){
         return "查找的部件不存在";
      }else{
         return part.getName()+"_"+part.getNumber();
      }

   }
   

}
```