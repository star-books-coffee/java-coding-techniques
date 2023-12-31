## 5장. 문제 발생에 대비하기
### 5.1.  빠른실패

```java
class CruiseControl {
    void setTargetSpeedKmh(double speedKmh) {
        if (speedKmh < 0) {
            throw new IllegalArgumentException();
        } else if (speedKmh <= SPEED_LIMIT) {
            targetSpeedKmh = speedKmh;
        } else {
            throw new IllegalArgumentException();
        }
    }
}
```

```java
class CruiseControl {
    void setTargetSpeedKmh(double speedKmh) {
        if (speedKmh < 0 || speedKmh > SPEED_LIMIT) {
            throw new IllegalArgumentException();
        }
        targetSpeedKmh = speedKmh;
    }
}
```

- **메서드를 빠르게 실패하도록 하라 (예외처리 로직을 먼저)**
- 매개변수 검증을 통한 예외처리를 우선적으로 하고, 일반적인 메서드 경로로 넘어갈 수 있게 구현하라

### 5.2. 항상 가장 구체적인 예외 잡기

- **예외를 잡으려면 가장 구체적인 예외 타입만 잡아라, 그렇지 않고 일반적인 타입을 잡으면 잡아선 안될 오류까지 잡힐 위험이 존재한다**
    - throwable 을 잡으면 OutOfMemoryError 와 같은 가상 머신 내 오류까지 잡힐 수 있음
    - Exception 대신 try 내 코드에서 던질 만한 가장 구체적 예외를 찾아라
- 가장 구체적인 예외를 잡기 위해 여러 예외를 잡아야 할 수도 있지만, 일반적 예외유형 잡는거보단 낫다

### 5.3. 메시지로 원인 설명

- 그냥 IllegalArgumentException() 이렇게 던지지 말고, **Exception 내에 메시지를 표시하라 (바라는 것, 받은 것, 전체 맥락)**
    - 세가지 정보는 나중에 `테스트 케이스` 로 재사용도 가능
### 5.4. 원인 사슬 깨지 않기

```java
class TransmissionParser {
	static Transmission parse(String rawMessage) {
		if (rawMessage != null) {
						&& rawMessage.length() != Transmission.MESSAGE_LENGTH) {
			throw new IllegalArgumentException(
					String.format("Expected %d, but got %d characters in '%s'", 
							Transmission.MESSAGE_LENGTH, rawMessage.length(),
							rawMessage));
		}

		try {
			...
		} catch (NumberFormatException e) {
				throw new IllegalArgumentException(
						String.format("Expected number, but got '%s' in '%s'",
										 rawId, rawMessage));
		}
	}
}
```

- 예외를 잡았지만 처리할 수 없다면 반드시 다시 던져야 함
    - 예외에 버그가 있을 경우 프로그램이 충돌할 때까지 전달될 수 있어야 함
- 위 코드는 NumberFormatException을 잡고도 유용한 메시지와 함께 IllegalArgumentException을 던지지만, **문제는 IllegalArgumentException 이 NumberFormatException 를 참조하지 않는다는 것**
    - IllegalArgumentException 의 스택 추적을 살펴보아도 NumberFormatException 에 관련된 정보가 없음
    - 즉, `원인 사슬` 이 끊어진 것 ; 예외가 그 예외를 일으킨 예외와 연결된 리스트를 보여주는 스택 추적을 확인할 수 없게 됨
- 원인 사슬이 깨지는 최악의 예시
    - NumberFormatException 이 throw 에서 빠짐으로써 원인만 연결되고 **예외 자체가 연결되지는 못함.**
    
    ```java
    } catch (NumberFormatException e) {
    	throw new IllegalArgumentException(e.getCause());
    }
    ```
    
- **catch 블록에서 예외를 던질 때는 message 와 잡았던 예외를 즉시 원인으로 전달하라**
    
    ```java
    throw new IllegalArgumentException("Message", e);
    ```
### 5.5. 변수로 원인 노출

```java
class TransmissionParser {
    static Transmission parse(String rawMessage) {
        if (rawMessage != null
                && rawMessage.length() != Transmission.MESSAGE_LENGTH) {
						
						// 1. before
            throw new IllegalArgumentException(
                String.format("Expected %d, but got %d characters **in '%s'**",
                    Transmission.MESSAGE_LENGTH, rawMessage.length(),
                    **rawMessage**));
						
						// after
            throw new MalformedMessageException(
                String.format("Expected %d, but got %d characters",
                    Transmission.MESSAGE_LENGTH, rawMessage.length()),
                    rawMessage);

        }

        String rawId = rawMessage.substring(0, Transmission.ID_LENGTH);
        String rawContent = rawMessage.substring(Transmission.ID_LENGTH);
        try {
            int id = Integer.parseInt(rawId);
            String content = rawContent.trim();
            return new Transmission(id, content);
        
				// 2. before
				} catch (NumberFormatException e) {
            throw new IllegalArgumentException(
                String.format("Expected number, but got '%s' **in '%s'**",
                    rawId, rawMessage), e);
        }

				// after
				} catch (NumberFormatException e) {
            throw new MalformedMessageException(
                String.format("Expected number, but got '%s'", rawId),
                rawMessage, e);
        }

    }
}
```

- IllegalArgumentException 의 message 에 두번이나 “in % s” 가 반복됨
- rawMessage 를 예외 메시지 String.format 내에 넣어서 출력하고 있기 때문에, **소프트웨어 최종 사용자가 어떤 종류의 메시지가 오류를 일으켰는지 알리고 싶을 때 추출하기 어려움**
→ **맞춤형 예외를 만들어, 그 예외만의 raw 메시지 필드가 들어간 예외를 정의하고 사용하라**

```java
final class MalformedMessageException extends IllegalArgumentException {
    final String raw;

    MalformedMessageException(String message, String raw) {
        super(String.format("%s in '%s'", message, raw));
        this.raw = raw;
    }
    MalformedMessageException(String message, String raw, Throwable cause) {
        super(String.format("%s in '%s'", message, raw), cause);
        this.raw = raw;
    }
}
```

- 예외 message 가 일관되게 유지됨
- 메시지 생성을 별도의 메서드로 추출하여 코드 중복도 제거됨

### 5.6. 타입 변환 전에 항상 타입 검증하기

```java
class Network {
    ...
    void listen() throws IOException, ClassNotFoundException {
        while (true) {
            Object signal = inputStream.readObject();
            CrewMessage crewMessage = (**CrewMessage**) signal;
            interCom.broadcast(crewMessage);
        }
    }
}
```

- 프로그램에서 동적 객체 사용시, 명시적으로 어떤 타입으로든 변환해야 함
    - 변환하지 않으면, RuntimeException 예외 발생 가능성이 존재
- 메서드는 스트림에 **실제로 어떤 타입이 들어올 지 제어할 수 없으므로**, 다른 타입이 들어오면 `ClassCastException` 이 발생
    - 예외를 잡을 수는 있지만, 전형적으로 수정 가능한 코드 내 버그를 알려주기 때문에 잡으면 안됨

```java
class Network {
		...
    void listen() throws IOException, ClassNotFoundException {
        while (true) {
            Object signal = inputStream.readObject();
            if (signal **instanceof CrewMessage**) {
                CrewMessage crewMessage = (CrewMessage) signal;
                interCom.broadcast(crewMessage);
            }
        }
    }
}
```

- **instanceof 연산자로 타입을 검증하라**
    - 여러 타입으로 된 메시지를 받고 싶으면 instanceof 로 차례대로 검증해야 함
- 프로그램이 외부와 상호작용할 때는, 항상 예상치 못한 입력을 처리할 수 있도록 대비해야 함
### 5.7. 항상 자원닫기

```java
DirectoryStream<Path> directoryStream = Files.newDirectoryStream(LOG_FOLDER, FILE_FILTER);

for (Path logFile : directoryStream) {
  result.add(logFile);
}

directoryStream.close();
```

- 더이상 자원이 필요없으면 바로 해제하면 된다
- 프로그램이 자원 연 후 close() 로 자원을 해제하기 전에, 예외가 발생하면 close() 가 실행되지 않아 프로그램 종료될 때까지 해제 X → `자원 누출`
    - 이후에 프로그램이 같은 자원 다시 요청시 프로그램 자체에도 문제 발생

```java
try (DirectoryStream<Path> directoryStream = Files.newDirectoryStream(LOG_FOLDER, FILE_FILTER)) {
	for (Path logFile : directoryStream) {
	  result.add(logFile);
	}
}
```

- **try-with-resources 구문 사용하라**, AutoCloseable 인터페이스를 구현한 클래스여야 동작함
- try 블록 끝나면 무슨 일이 있어도 자바가 알아서 close() 호출 처리
- 컴파일러가 아래처럼 확장하여 동작
    - finally 블록에서 자원을 닫되, null 이 아닐 때만 닫음으로써 NPE 피함

```java
DirectoryStream<Path> directoryStream = Files.newDirectoryStream(LOG_FOLDER, FILE_FILTER);
try {
	// 자원사용
} finally {
	if(resource != null) {
		resource.close();
	}
}
```

### 5.8. 항상 다수 자원닫기

```java
try (DirectoryStream<Path> directoryStream = Files.newDirectoryStream(LOG_FOLDER, FILE_FILTER); BufferedWriter writer = Files.newBufferedWriter(STATISTICS_CSV)) { ... }
```

- **여러 자원을 사용하고 싶다면, try-with-resources 안에서 세미콜론으로 구분해줘라**
- 내부적으로 컴파일러가 아래처럼 확장하여 동작
    - try-with-resources 블록 내 각 자원을 확장해 여러 중첩 블록 생성

```java
// resource1 열기
try {
	// resource2 열기 
	try { 
		// resource1, resource2 사용
	} finally {
		resource2.close();
} finally {
	resource1.close();
}
```

### 5.9. 빈 catch 블록 설명하기

```java
try (DirectoryStream<Path> directoryStream = Files.newDirectoryStream(LOG_FOLDER, FILE_FILTER)) {
		for (Path logFile : directoryStream) {
	    result.add(logFile);
    }
} catch (NotDirectoryException e) {

}
```

- **예외는 의미있게 처리할 수 있을 때만 잡아야 한다.**
- 빈 catch 블록은 무조건 버그처럼 보인다

```java
try (DirectoryStream<Path> directoryStream = Files.newDirectoryStream(LOG_FOLDER, FILE_FILTER)) {
		for (Path logFile : directoryStream) {
	    result.add(logFile);
    }
} catch (NotDirectoryException ignored) {
// No directory -> no logs!
}
```

- **예외 변수명을 e → ignored 로 바꾸어, 예외를 무시하겠다고 명시적으로 드러내라**
- **예외를 왜 무시하는지 주석으로 추가하라**
