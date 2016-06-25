---
layout: post
title: JAVA의 직렬화와 serialVersionUID
categories: [java]
tags: [java]
description: java serialize
comments: true
---

# Java의 직렬화와  serialVersionUID란?

얼마전 serialVersionUID에 대한 간단한 설명을 듣고, 그 동안 보면서도 아무 생각없이 지나갔던 자바의 직렬화와 serialVersionUID에 대해 알아봐야 겠다는 생각을 했다.  

## Serialize란?

우선, serialize가 무엇인지에 대한 개념부터 간단하게 집고 넘어가자.  
내가 처음으로 직렬화를 접한 것은 학교 수업도중 자바를 이용해서 그림판을 만들었을때였다. 내가 작업중인 화면을 파일로 저장하는 기능을 만들어야 했는데, 그림판의 요소인 도형이나 라인들을 저장하려면 객체자체를 파일로 저장하고 불러올 수 있어야 했다.  

이럴때 자바에서 제공하는 **Serializable** 인터페이스를 이용하면 객체의 상태를 바이트의 형태로 만들어 파일로 저장하거나, 네트워크 통신을 할 수 있다.  
**Serializable**는 구현해야할 메서드나 필드가 없다. Java에게 직렬화 가능한 클래스라는 사실을 알려주는 역할만 한다.

참고로 직렬화 객체는 접근 가능한 default생성자가 있어야 한다.

### 직렬화 과정에서 특별한 작업을 하고 싶은 경우 ?!

개발 과정에서 직렬화 중간에 특별한 작업을 추가하고 싶은 경우가 있을 수 있다.  이런 경우를 위해 자바에서는 특별한 메서드를 제공한다.  
다음 메서드들이 바로 serializable을 구현한 클래스가 메서드를 만들면 직렬화 과정에서 특별한 작업을 할 수 있도록 도와주는 메서드들이다.  

```java
 private void writeObject(java.io.ObjectOutputStream out)
     throws IOException
 private void readObject(java.io.ObjectInputStream in)
     throws IOException, ClassNotFoundException;
 private void readObjectNoData()
     throws ObjectStreamException;
 ANY-ACCESS-MODIFIER Object writeReplace() throws ObjectStreamException;
 ANY-ACCESS-MODIFIER Object readResolve() throws ObjectStreamException;
```

특이한 점은 위에서 언급한 것 처럼 `Serializable은 구현해야 할 메서드가 없다`라는 특징이 있기 때문에 오버라이딩이 아니라는 점이다.

클래스가 위 메서드들을 구현하게 되면 기본으로 serialize/desirialize 과정을 처리해주는 기능을 사용할 수 없기 때문에 직접 직렬화 과정을 처리해야 한다. 다행히도 상위/하위 클래스의 필드까지 신경쓸 필요는 없는 것 같다.

그럼, 어떻게 변하는지 writeObject로 한가지 예시만 살펴보자.

```java
public class SerializableObject implements Serializable {
	
	String data;
	transient String data2;
	
	public SerializableObject(String data, String data2)	{
		this.data = data;
		this.data2 = data2;
	}
}
```

```java
public class App {
	
	private static final String filename = "objectData.dat";
    public static void main( String[] args ) {
    	
    	SerializableObject o1 = new SerializableObject("1", "2");
    	SerializableObject o2 = new SerializableObject("3", "4");
    	
    	saveObject(o1,o2);
    	
    	List objectList = readObject();
    	
    	for(Object o : objectList) {
    		SerializableObject serializableObject = (SerializableObject)o;
    		System.out.println(serializableObject.data);
    		System.out.println(serializableObject.data2);
    	}
    }
    
    private static void saveObject(Object... objects) {
    	
    	try(FileOutputStream fileOutputStream = new FileOutputStream(filename);
        	ObjectOutputStream objectOutputStream = new ObjectOutputStream(fileOutputStream);) {
        	
    		for(Object o : objects) {
	        	objectOutputStream.writeObject(o);
    		}
        	
        } catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
    }
    
    private static List readObject() {
    	List objectList = new ArrayList<Object>();
    	try(FileInputStream fileInputStream = new FileInputStream(filename);
    		ObjectInputStream objectInputStream = new ObjectInputStream(fileInputStream);) {
    		
    		Object readObject;
    		while((readObject = objectInputStream.readObject()) != null) {
    			objectList.add(readObject);
    		}
    		
    	} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} catch (ClassNotFoundException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
    	
    	return objectList;
    }
}

```

위 코드를 실행했을 경우의 결과는 아래 그림과 같이 나온다.  
**transient**의 경우 직렬화 대상에서 제외되기 때문에 readObject로 읽어온 객체에서는 null로 나타다는 것을 볼 수 있다.  

![Image alt]({{ site.baseurl }}/assets/img/serialize2.PNG "실행결과1")  

위 메서드들 중 writeObject를 클래스에 작성하게 되면 어떤 결과가 나오는지 확인해보자. SSerializableObject만 아래와 같이 변경하였다.

```java
public class SerializableObject extends SerializableParentObject {
	
	String data;
	transient String data2;
	
	public SerializableObject(String data, String data2)	{
		this.data = data;
		this.data2 = data2;
	}

	private void writeObject(java.io.ObjectOutputStream out)
		     throws IOException {
		 System.out.println("WriteObject method call");
		 
	 }
}
```

결과는 다음과 같이 제대로 직렬화가 이뤄지지 않아서 읽어오는 부분에서 에러가 발생하는 것을 확인할 수 있다.  

![Image alt]({{ site.baseurl }}/assets/img/serialize3.PNG "실행결과2")  

정리하자면  
1. 객체를 직렬화 하고 싶으면 Serializable인터페이스를 implements하라.
2. Serializable은 별다른 메서드와 프로퍼티를 갖고 있지 않다.
3. 그러나 지정된 형식의 메서드를 구현함으로써, 직렬화 과정에서 원하는 동작을 하도록 할 수 있다.
4. 위 3번의 경우 기본으로 직렬화 하는 동작을 하지 않으니 주의해야 한다.

## 그럼 serialVersionUID는 뭐지?

클래스에 Serializable 인터페이스를 implements하다보면, 아래와 같은 경고 메시지를 마주하게 된다. 이런 경우, 이클립스가 랜덤으로 생성해주는 serialVersionUID를 사용하거나 warning으로 나오는 경고를 무시하고 진행한 적이 있다.  

![Image alt]({{ site.baseurl }}/assets/img/serialize1.png "warning메시지")  

도대체 serialVersionUID가 무엇이길래 알 수 없는 숫자를 선언하도록 권유하는 것일까?  
이름에서도 대충 눈치챌 수 있겠지만, 직렬화를 하는 대상에서 사용하는 고유한 ID로 생각하면 될 것 같다. serialize한 객체를 deserialize할 경우 같은 내용을 가진 클래스라도 serialVersionUID가 다르게 되면 deserialize할 수 없는 클래스로 판단하고 InvalidClassException을 발생시킨다.

직렬화/ 역직렬화 과정이 필요한데 serialVersionUID를 지정해주지 않았다면, JVM은 환경에 따라 임의로 생성하게 될 것이고 serialize 할때의 값과 deserialize할때의 값이 다르다면 문제가 발생할 것이다.


## 주의할 점
http://egloos.zum.com/dojeun/v/317825  
블로그를 보다가 알게 된 사실인데, ObjectOutputStream는 대상 object를 캐시해놓기 때문에 같은 객체의 프로퍼티를 변경해도 제대로 writeObject가 이뤄지지 않을 수 있다. 이를 방지하기 위해 reset()메서드를 호출해줘야 한다.

```java
SerializableObject o = new SerializableObject();
o.a = 1;
objectOutputStream.writeObject(o);
serializableObject.a++;
objectOutputStream.writeObject(o);
```

위 코드를 실행했을 경우 3번재 줄과 5번째 줄의 출력값이 서로 다르기를 기대하겠지만, 실제로는 둘다 a의 값을 1로 출력하고 있다.

## 참고
https://docs.oracle.com/javase/7/docs/api/java/io/Serializable.html
http://egloos.zum.com/dojeun/v/317825
https://docs.oracle.com/javase/7/docs/platform/serialization/spec/serial-arch.html