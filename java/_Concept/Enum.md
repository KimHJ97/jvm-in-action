# Java Enum

Enum은 열거형 상수의 집합을 나타내는 특별한 데이터 타입이다.

Enum을 사용하면 코드가 더 명확해지고, 컴파일 타임에 타입 체크가 가능하여 런타임 오류를 줄일 수 있다.

 - 타입 안전성
 - 싱글톤 보장: 각 열거 상수는 단 하나의 인스턴스만 존재
 - 기본 메서드 제공: values(), valueOf(), name(), ordinal() 메서드 자동 생성
 - Comparable 인터페이스 구현: 열거 상수 간 순서 비교 가능

## 1. Enum 사용법

 - ordinal() 메서드는 Enum 상수의 순서가 바뀌면 값이 바뀌기 떄문에 비즈니스 로직에서 사용하는 것은 피하는 것이 좋다.
```java
public class DayEnumUsage {
    // Enum 정의
    enum Day {
        MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
    }
    
    // Enum 사용
    public static void main(String[] args) {
        Day today = Day.MONDAY;
        
        // 모든 열거 상수 가져오기: [MONDAY, TUESDAY, ...]
        Day[] allDays = Day.values();
        for (Day day : allDays) {
            System.out.println(day);
        }
        
        // 문자열로부터 열거 상수 가져오기: Day.TUESDAY
        Day tuesday = Day.valueOf("TUESDAY");
        
        // 열거 상수의 이름 가져오기: "MONDAY"
        String name = Day.MONDAY.name(); 
        
        // 열거 상수의 순서 가져오기 (0부터 시작)
        int ordinal = Day.WEDNESDAY.ordinal();  // 2
    }
}
```

 - `Swtich문 활용`
```java
Day today = Day.MONDAY;

// Switch문
switch (today) {
    case MONDAY:
        System.out.println("월요일");
        break;
    case FRIDAY:
        System.out.println("금요일");
        break;
    case SATURDAY:
    case SUNDAY:
        System.out.println("주말");
        break;
    default:
        System.out.println("평일");
}

// Switch 표현식 (JDK14)
String message = switch (today) {
    case MONDAY -> "월요병";
    case FRIDAY -> "금요일";
    case SATURDAY, SUNDAY -> "주말";
    default -> "평일";
};
```

 - `필드와 생성자 추가`
```java
// Enum 정의
@Getter
@AllArgsConstructor
public enum Planet {
    MERCURY(3.303e+23, 2.4397e6),
    VENUS(4.869e+24, 6.0518e6),
    EARTH(5.976e+24, 6.37814e6),
    MARS(6.421e+23, 3.3972e6),
    JUPITER(1.9e+27, 7.1492e7),
    SATURN(5.688e+26, 6.0268e7),
    URANUS(8.686e+25, 2.5559e7),
    NEPTUNE(1.024e+26, 2.4746e7);

    private final double mass;   // 질량 (kg)
    private final double radius; // 반지름 (m)

    // 표면 중력 계산 메서드
    public double surfaceGravity() {
        double G = 6.67300E-11; // 중력 상수
        return G * mass / (radius * radius);
    }
}

// Enum 사용
Planet earth = Planet.EARTH;
System.out.println("지구의 질량: " + earth.getMass() + " kg");
System.out.println("지구의 반지름: " + earth.getRadius() + " m");
System.out.println("지구의 표면 중력: " + earth.surfaceGravity() + " m/s²");
```

 - `메서드 오버라이딩`
```java
// Enum 정의
public enum Operation {
    PLUS {
        public double apply(double x, double y) { return x + y; }
    },
    MINUS {
        public double apply(double x, double y) { return x - y; }
    },
    TIMES {
        public double apply(double x, double y) { return x * y; }
    },
    DIVIDE {
        public double apply(double x, double y) { return x / y; }
    };

    // 추상 메서드 선언 (각 상수가 구현해야 함)
    public abstract double apply(double x, double y);
}

// Enum 사용
double result = Operation.PLUS.apply(10, 5);  // 15.0
System.out.println(Operation.MINUS.apply(10, 5));  // 5.0
System.out.println(Operation.TIMES.apply(10, 5));  // 50.0
System.out.println(Operation.DIVIDE.apply(10, 5)); // 2.0
```

 - `인터페이스 구현`
```java
public class Test {

    public static void main(String[] args) {
        Season spring = Season.SPRING;
        System.out.println(spring.getDescription());
    }
    
    // 인터페이스 정의
    interface Describable {
        String getDescription();
    }

    // Enum 정의
    @Getter
    @AllArgsConstructor
    enum Season implements Describable {
        SPRING("봄") {
            @Override
            public String getDescription() {
                return super.getSeason() + ": 꽃이 피는 계절";
            }
        },
        SUMMER("여름") {
            @Override
            public String getDescription() {
                return super.getSeason() + ": 더운 계절";
            }
        },
        FALL("가을") {
            @Override
            public String getDescription() {
                return super.getSeason() + ": 단풍이 드는 계절";
            }
        },
        WINTER("겨울") {
            @Override
            public String getDescription() {
                return super.getSeason() + ": 추운 계절";
            }
        };

        private final String season;
    }

}
```

 - `EnumSet과 EnumMap`
    - Java는 Enum을 위한 특별한 컬렉션 클래스를 제공
    - EnumSet은 Enum 상수를 위한 고성능 Set 구현체
    - EnumMap은 Enum을 키로 사용하는 특화된 Map 구현체
    - EnumSet과 EnumMap은 내부적으로 비트 벡터나 배열을 사용하기 때문에 메모리 사용량이 적고 연산 속도가 빨라 일반 HashSet이나 HashMap보다 훨씬 효율적이다.
```java
// 1. EnumSet 사용법
// 모든 요일 포함
EnumSet<Day> allDays = EnumSet.allOf(Day.class);

// 빈 EnumSet 생성
EnumSet<Day> noDays = EnumSet.noneOf(Day.class);

// 특정 요일만 포함
EnumSet<Day> workDays = EnumSet.of(Day.MONDAY, Day.TUESDAY, Day.WEDNESDAY, Day.THURSDAY, Day.FRIDAY);

// 범위로 지정
EnumSet<Day> weekend = EnumSet.range(Day.SATURDAY, Day.SUNDAY);

// 차집합
EnumSet<Day> notWorkDays = EnumSet.complementOf(workDays);


// 2. EnumMap 사용법
// EnumMap 생성
EnumMap<Day, String> daySchedule = new EnumMap<>(Day.class);

// 값 추가
daySchedule.put(Day.MONDAY, "주간 회의");
daySchedule.put(Day.WEDNESDAY, "프로젝트 발표");
daySchedule.put(Day.FRIDAY, "팀 점심");

// 값 가져오기
System.out.println(daySchedule.get(Day.MONDAY));  // "주간 회의"
```

## 2. Enum 사용 예시

 - `상태 관리 (State Management)`
   - 각 상태에서 다음 상태로 전이할 수 있는지 검사하는 로직을 Enum 내부에 캡슐화
```java
@AllArgsConstructor
@Getter
public enum OrderStatus {
   PENDING("대기 중", "주문이 접수되었지만 아직 처리되지 않았습니다."),
   PROCESSING("처리 중", "주문이 현재 처리 중입니다."),
   SHIPPED("배송 중", "상품이 배송 중입니다."),
   DELIVERED("배송 완료", "상품이 성공적으로 배송되었습니다."),
   CANCELLED("취소됨", "주문이 취소되었습니다.");

   private final String displayName;
   private final String description;

   // 상태 전이가 가능한지 확인하는 메서드
   public boolean canTransitionTo(OrderStatus nextStatus) {
      switch (this) {
         case PENDING:
            return nextStatus == PROCESSING || nextStatus == CANCELLED;
         case PROCESSING:
            return nextStatus == SHIPPED || nextStatus == CANCELLED;
         case SHIPPED:
            return nextStatus == DELIVERED || nextStatus == CANCELLED;
         case DELIVERED:
         case CANCELLED:
            return false;
         default:
            throw new IllegalStateException("Unknown order status: " + this);
      }
   }
}
```

 - `권한 관리 (Permission Management)`
   - 비트 플래그(Bit Flags)를 사용해 여러 권한을 하나의 정수에 효율적으로 저장
```java
@Getter
@AllArgsConstructor
public enum Permission {
   READ(1),      // 2^0 = 1
   WRITE(2),     // 2^1 = 2
   EXECUTE(4),   // 2^2 = 4
   DELETE(8),    // 2^3 = 8
   ADMIN(16);    // 2^4 = 16

   private final int bit;

   // 권한 집합에 특정 권한이 포함되어 있는지 확인
   public static boolean hasPermission(int permissionSet, Permission permission) {
      return (permissionSet & permission.getBit()) != 0;
   }

   // 권한 집합에 새 권한 추가
   public static int addPermission(int permissionSet, Permission permission) {
      return permissionSet | permission.getBit();
   }

   // 권한 집합에서 특정 권한 제거
   public static int removePermission(int permissionSet, Permission permission) {
      return permissionSet & ~permission.getBit();
   }
}

// 초기 권한 설정 (읽기 + 쓰기)
int userPermissions = 0;
userPermissions = Permission.addPermission(userPermissions, Permission.READ);
userPermissions = Permission.addPermission(userPermissions, Permission.WRITE);

// 권한 확인
System.out.println("읽기 권한: " + Permission.hasPermission(userPermissions, Permission.READ));        // true
System.out.println("쓰기 권한: " + Permission.hasPermission(userPermissions, Permission.WRITE));       // true
System.out.println("실행 권한: " + Permission.hasPermission(userPermissions, Permission.EXECUTE));     // false

// 권한 추가
userPermissions = Permission.addPermission(userPermissions, Permission.EXECUTE);
System.out.println("실행 권한: " + Permission.hasPermission(userPermissions, Permission.EXECUTE));     // true

// 권한 제거
userPermissions = Permission.removePermission(userPermissions, Permission.WRITE);
System.out.println("쓰기 권한: " + Permission.hasPermission(userPermissions, Permission.WRITE));       // false
```

 - `HTTP 상태 코드 관리`
   - HTTP 상태 코드와 관련된 정보와 로직을 Enum에 캡슐화
```java
@Getter
@AllArgsConstructor
public enum HttpStatus {
    // 2xx: 성공
    OK(200, "OK"),
    CREATED(201, "Created"),
    ACCEPTED(202, "Accepted"),
    NO_CONTENT(204, "No Content"),
    
    // 4xx: 클라이언트 오류
    BAD_REQUEST(400, "Bad Request"),
    UNAUTHORIZED(401, "Unauthorized"),
    FORBIDDEN(403, "Forbidden"),
    NOT_FOUND(404, "Not Found"),
    METHOD_NOT_ALLOWED(405, "Method Not Allowed"),
    
    // 5xx: 서버 오류
    INTERNAL_SERVER_ERROR(500, "Internal Server Error"),
    NOT_IMPLEMENTED(501, "Not Implemented"),
    BAD_GATEWAY(502, "Bad Gateway"),
    SERVICE_UNAVAILABLE(503, "Service Unavailable");

    private final int code;
    private final String message;

    // 코드로 상태 찾기
    public static HttpStatus valueOf(int statusCode) {
        for (HttpStatus status : values()) {
            if (status.code == statusCode) {
                return status;
            }
        }
        throw new IllegalArgumentException("Invalid HTTP status code: " + statusCode);
    }

    // 성공 상태인지 확인
    public boolean isSuccess() {
        return code >= 200 && code < 300;
    }

    // 클라이언트 오류인지 확인
    public boolean isClientError() {
        return code >= 400 && code < 500;
    }

    // 서버 오류인지 확인
    public boolean isServerError() {
        return code >= 500 && code < 600;
    }
}

// 상태 코드 사용
HttpStatus status = HttpStatus.OK;
System.out.println("Status: " + status.getCode() + " " + status.getMessage());

// 코드로 상태 찾기
HttpStatus notFound = HttpStatus.valueOf(404);
System.out.println("Found status: " + notFound.name());

// 상태 분류 확인
System.out.println("Is success: " + status.isSuccess());
System.out.println("Is client error: " + notFound.isClientError());
```

## 3. Enum과 디자인 패턴

 - `전략 패턴 (Strategy Pattern)`
   - 전략 패턴은 알고리즘을 캡슐화하고 실행 시점에 알고리즘을 선택할 수 있게 해주는 패턴이다.
   - 결제 방법의 로직이 한 곳에 모여있어 유지보수가 용이.
   - 새로운 결제 방법을 추가할 때 Enum에 새 상수만 추가하면 되므로 확정성이 좋음.
```java
public enum PaymentStrategy {
    CREDIT_CARD {
        @Override
        public void pay(double amount) {
            System.out.println("신용카드로 " + amount + "원 결제");
            // 신용카드 결제 로직
        }
    },
    PAYPAL {
        @Override
        public void pay(double amount) {
            System.out.println("페이팔로 " + amount + "원 결제");
            // 페이팔 결제 로직
        }
    },
    BANK_TRANSFER {
        @Override
        public void pay(double amount) {
            System.out.println("계좌이체로 " + amount + "원 결제");
            // 계좌이체 결제 로직
        }
    },
    MOBILE_PAYMENT {
        @Override
        public void pay(double amount) {
            System.out.println("모바일 결제로 " + amount + "원 결제");
            // 모바일 결제 로직
        }
    };

    public abstract void pay(double amount);
}

// 결제 방법 선택
PaymentStrategy paymentMethod = PaymentStrategy.CREDIT_CARD;

// 결제 실행
paymentMethod.pay(50000);
```

 - `싱글톤 패턴 (Singleton Pattern)`\
   - Enum은 기본적으로 싱글톤 패턴을 구현한다. 각 Enum 상수는 단 하나의 인스턴스만 존재한다.
   - 전통적인 싱글톤 구현보다 직렬화, 리플렉션 공격에 안전하고, 코드도 더 간결해진다.
```java
@Getter
public enum DatabaseConnection {
    INSTANCE;
    
    private Connection connection;
    
    // 초기화 블록
    DatabaseConnection() {
        try {
            // 데이터베이스 연결 설정
            String url = "jdbc:mysql://localhost:3306/mydb";
            String user = "username";
            String password = "password";
            connection = DriverManager.getConnection(url, user, password);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
    
    public void executeQuery(String query) {
        try {
            Statement stmt = connection.createStatement();
            ResultSet rs = stmt.executeQuery(query);
            // 결과 처리
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}

// 어디서든 동일한 인스턴스에 접근
DatabaseConnection db = DatabaseConnection.INSTANCE;
db.executeQuery("SELECT * FROM users");
```

 - `팩토리 패턴 (Factory Pattern)`
   - 객체 생성 로직을 캡슐화하고, 타입 안전성을 제공한다.
```java
// 공통 인터페이스
interface Vehicle {
    void drive();
}

// 구현 클래스들
class Car implements Vehicle {
    @Override
    public void drive() {
        System.out.println("자동차를 운전합니다.");
    }
}

class Motorcycle implements Vehicle {
    @Override
    public void drive() {
        System.out.println("오토바이를 운전합니다.");
    }
}

class Bicycle implements Vehicle {
    @Override
    public void drive() {
        System.out.println("자전거를 탑니다.");
    }
}

// Enum 팩토리
public enum VehicleFactory {
    CAR {
        @Override
        public Vehicle create() {
            return new Car();
        }
    },
    MOTORCYCLE {
        @Override
        public Vehicle create() {
            return new Motorcycle();
        }
    },
    BICYCLE {
        @Override
        public Vehicle create() {
            return new Bicycle();
        }
    };
    
    public abstract Vehicle create();
    
    // 문자열로부터 팩토리 가져오기
    public static VehicleFactory getFactory(String vehicleType) {
        try {
            return valueOf(vehicleType.toUpperCase());
        } catch (IllegalArgumentException e) {
            throw new IllegalArgumentException("지원하지 않는 차량 유형: " + vehicleType);
        }
    }
}

// 직접 팩토리 사용
Vehicle car = VehicleFactory.CAR.create();
car.drive();  // "자동차를 운전합니다."

// 문자열로부터 팩토리 가져오기
String vehicleType = "bicycle";
Vehicle vehicle = VehicleFactory.getFactory(vehicleType).create();
vehicle.drive();  // "자전거를 탑니다."
```

 - `명령 패턴 (Command Pattern)`
```java
public enum TextEditorCommand {
    COPY {
        @Override
        public void execute(TextEditor editor) {
            editor.copy();
        }
    },
    CUT {
        @Override
        public void execute(TextEditor editor) {
            editor.cut();
        }
    },
    PASTE {
        @Override
        public void execute(TextEditor editor) {
            editor.paste();
        }
    },
    UNDO {
        @Override
        public void execute(TextEditor editor) {
            editor.undo();
        }
    },
    REDO {
        @Override
        public void execute(TextEditor editor) {
            editor.redo();
        }
    };
    
    public abstract void execute(TextEditor editor);
}

// 텍스트 에디터 클래스
class TextEditor {
    public void copy() { System.out.println("텍스트 복사"); }
    public void cut() { System.out.println("텍스트 잘라내기"); }
    public void paste() { System.out.println("텍스트 붙여넣기"); }
    public void undo() { System.out.println("실행 취소"); }
    public void redo() { System.out.println("다시 실행"); }
}

TextEditor editor = new TextEditor();

// 명령 실행
TextEditorCommand.COPY.execute(editor);  // "텍스트 복사"
TextEditorCommand.PASTE.execute(editor); // "텍스트 붙여넣기"

// 단축키 매핑 예시
Map<String, TextEditorCommand> shortcuts = new HashMap<>();
shortcuts.put("Ctrl+C", TextEditorCommand.COPY);
shortcuts.put("Ctrl+X", TextEditorCommand.CUT);
shortcuts.put("Ctrl+V", TextEditorCommand.PASTE);
shortcuts.put("Ctrl+Z", TextEditorCommand.UNDO);
shortcuts.put("Ctrl+Y", TextEditorCommand.REDO);

// 단축키 처리
String input = "Ctrl+V";
if (shortcuts.containsKey(input)) {
    shortcuts.get(input).execute(editor);  // "텍스트 붙여넣기"
}
```

## 4. 성능과 메모리 고려사항

 - `메모리 사용량`
   - Enum은 기본적으로 싱글톤 인스턴스로 궇녀되어 메모리 사용량이 적다. 하지만, Enum 상수가 많아지거나, 각 상수에 많은 필드를 추가하면 메모리 사용량이 증가한다.
   - 대부분의 애플리케이션에서는 Enum의 메모리 사용량이 문제가 되지 않지만, 메모리가 제한된 환경(예: 임베디드 시스템)에서는 Enum 대신 상수를 사용하는 것이 더 효율적일 수 있다.
   - 기본 Enum (상수만): 매우 적은 메모리 사용 
   - 필드가 있는 Enum: 각 필드마다 추가 메모리 사용 
   - 메서드가 있는 Enum: 클래스 정보에 추가 메모리 사용 
   - 내부 클래스가 있는 Enum: 더 많은 클래스 정보에 추가 메모리 사용
 - `초기화 비용`
   - Enum은 클래스가 로드될 떄 모든 상수 인스턴스가 생성된다.
   - 생성자에서 복잡한 작업(DB 연결, 파일 I/O)을 수행하면 초기화 시간이 길어진다.
   - 때문에, Enum 생성자에서는 가벼운 작업만 수행하고, 무거운 작업은 지연 초기화를 사용하는 것을 권장
   - 지연 초기화를 사용하면 Enum 상수들은 즉시 초기화되지만, 내부 리소스들은 해당 리소스가 처음 호출될 때만 초기화된다.
```java
public enum ResourceManager {
    INSTANCE;
    
    private Resource resource;
    
    // 지연 초기화를 위한 메서드
    public Resource getResource() {
        if (resource == null) {
            synchronized (this) {
                if (resource == null) {
                    resource = new Resource();
                }
            }
        }
        return resource;
    }
    
    // 리소스 클래스
    private static class Resource {
        public Resource() {
            System.out.println("무거운 리소스 초기화 중...");
            try {
                Thread.sleep(1000); // 시뮬레이션
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("리소스 초기화 완료!");
        }
        
        public void use() {
            System.out.println("리소스 사용 중...");
        }
    }
}
```

## 5. Enum 주의 사항

 - `상속 불가능`
   - Enum은 java.lang.Enum 클래스를 암시적으로 상속받기 때문에, 다른 클래스를 상속받을 수 없다.
   - 이 제한을 우회하려면 인터페이스를 사용하거나, 합성을 활용해야 한다.
```java
// 인터페이스 구현
public interface MyInterface {
    void doSomething();
}

public enum MyEnum implements MyInterface {
    INSTANCE_1 {
        @Override
        public void doSomething() {
            // 구현
        }
    },
    INSTANCE_2 {
        @Override
        public void doSomething() {
            // 구현
        }
    };
}

// 합성 활용
public class MyClass {
    public void helperMethod() {
        // 유용한 기능
    }
}

public enum MyEnum {
    INSTANCE_1, INSTANCE_2;
    
    private final MyClass helper = new MyClass();
    
    public void doSomething() {
        helper.helperMethod();
        // 추가 로직
    }
}
```

 - `리플렉션과 Enum`
   - Enum은 기본적으로 싱글톤 패턴으로 구현된다.
   - Enum은 리플렉션을 통해 새 인스턴스를 생성할 수 없다. (리플렉션 공격에 안전)
```java
// 실패하는 코드
try {
    Constructor<Day> constructor = Day.class.getDeclaredConstructor();
    constructor.setAccessible(true);
    Day day = constructor.newInstance();  // ❌ IllegalArgumentException 발생
} catch (Exception e) {
    e.printStackTrace();
}
```

 - `메모리 누수 가능성`
   - Enum 상수는 JVM이 종료될 때까지 GC되지 않기 때문에, Enum 상수가 무거운 객체(예: 대용량 컬렉션, 파일 핸들)를 참조하면 메모리 누수가 발생할 수 있다.
   - 
```java
// ❌ 잠재적 메모리 누수
public enum ResourceHolder {
    INSTANCE;
    
    private final List<byte[]> largeData;
    
    ResourceHolder() {
        largeData = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            largeData.add(new byte[1024 * 1024]); // 1MB 배열 1000개 = 약 1GB
        }
    }
}

// ✔ 지연 초기화로 개선
public enum ResourceHolder {
    INSTANCE;
    
    private List<byte[]> largeData;
    
    public List<byte[]> getLargeData() {
        if (largeData == null) {
            largeData = new ArrayList<>();
            for (int i = 0; i < 1000; i++) {
                largeData.add(new byte[1024 * 1024]);
            }
        }
        return largeData;
    }
    
    public void clearData() {
        largeData = null;
    }
}
```
