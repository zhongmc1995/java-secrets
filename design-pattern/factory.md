# 工厂模式
## 普通工厂模式
普通工厂模式，就是建立一个工厂类，对实现了同一接口的产品类进行实例的创建。
## 示例代码
```java
//发送短信和邮件的接口
public interface Sender {  
    public void Send();  
} 

//发送邮件的实现类
public class MailSender implements Sender {  
    public void Send() {  
        System.out.println("发送邮件!");  
    }  
}  
//发送短信的实现类
public class SmsSender implements Sender {  
    public void Send() {  
        System.out.println("发送短信!");  
    }  
}  

//创建工厂类
public class SendFactory {  
    //工厂方法  普通工厂方法
    public Sender produce(String type) {  
        if ("mail".equals(type)) {  
            return new MailSender();  
        } else if ("sms".equals(type)) {  
            return new SmsSender();  
        } else {  
            System.out.println("请输入正确的类型!");  
            return null;  
        }  
    }  
} 	

//测试类
public class FactoryTest {  

    public static void main(String[] args) {  
        SendFactory factory = new SendFactory();  
        Sender sender = factory.produce("sms");
        sender.Send();  
    }  
```
## 多个工厂方法
多个工厂方法模式 是对普通工厂方法模式的改进，在普通工厂方法模式中，如果传递的字符串出错，则不能正确创建对象，而多个工厂方法模式是提供多个工厂方法，分别创建对象。
## 示例代码
```java
//将上面的代码做下修改，改动下SendFactory类就行
//这个就不用根据用户传的字符串类创建对象了
public class SendFactory {  

  //多方法工厂模式
    public Sender produceMail(){  
        return new MailSender();  
    }  

    public Sender produceSms(){  
        return new SmsSender();  
    }  
}

//测试类
public class FactoryTest {  

    public static void main(String[] args) {  
        SendFactory factory = new SendFactory();  
        Sender sender = factory.produceMail();  
        sender.Send();  
    }  
}
  ```
## 静态工厂方法
静态工厂方法模式，将上面的多个工厂方法模式里的方法置为静态的，不需要创建实例，直接调用即可。
## 示例代码
 ```java
public class SendFactory {  

    public static Sender produceMail(){  
        return new MailSender();  
    }  

    public static Sender produceSms(){  
        return new SmsSender();  
    }  
}  

//测试类
public class FactoryTest {  

    public static void main(String[] args) {      
        Sender sender = SendFactory.produceMail();  
        sender.Send();  
    }  
}
  ```
## 抽象工厂
工厂方法模式有一个问题就是，类的创建依赖工厂类，也就是说，如果想要拓展程序，必须对工厂类进行修改，这违背了闭包原则，所以，从设计角度考虑，有一定的问题，如何解决？就用到抽象工厂模式，创建多个工厂类，这样一旦需要增加新的功能，直接增加新的工厂类就可以了，不需要修改之前的代码。
## 示例代码
```java
//发送短信和邮件的接口
public interface Sender {  
    public void Send();  
} 

//发送邮件的实现类
public class MailSender implements Sender {  
    public void Send() {  
        System.out.println("发送邮件!");  
    }  
}  
//发送短信的实现类
public class SmsSender implements Sender {  
    public void Send() {  
        System.out.println("发送短信!");  
    }  
}  
public class WX imp Sender{
    public void send(){
  syso(“微信发送”);
  }
}
public class SendWXFactory implements Provider {  
    public Sender produce(){  
        return new WX();  
    }  
}  

//给工厂类一个接口
public interface Provider {  
    public Sender produce();  
}  
//两个工厂的实现类
public class SendMailFactory implements Provider {  
    public Sender produce(){  
        return new MailSender();  
    }  
}  


//测试类
public class Test {  

    public static void main(String[] args) {  
        Provider provider = new SendMailFactory();  
        Sender sender = provider.produce();  
        sender.Send();  
    }  
}
```
**注**:这个模式的好处就是，如果你现在想增加一个功能：发送及时信息，则只需做一个实现类实现Sender接口，同时做一个工厂类，实现Provider接口，就OK了，无需去改动现成的代码。这样做，拓展性较好
