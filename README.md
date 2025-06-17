# Specification-And-Criteria

# Java və Spring Boot-da Specification və Criteria API: Dərin Bələdçi (2025)

Bu sənəd Java-da, xüsusilə Spring Data JPA ilə istifadə olunan **Specification** interfeysi və **Criteria API** haqqında ətraflı məlumat təqdim edir. Hər iki mexanizmin təyinatı, işləmə prinsipi, Spring Boot-da istifadə ssenariləri, üstünlükləri, çətinlikləri və praktiki nümunələr detallı şəkildə izah olunur. Sənəd 2025-ci ilə aktual olaraq Spring Boot 3.2+ versiyalarına əsaslanır.

---

## 1. Specification Nədir?

**Specification** Spring Data JPA-da dinamik və təkrar istifadə edilə bilən sorğular yaratmaq üçün istifadə olunan bir interfeysdir. `org.springframework.data.jpa.domain.Specification` interfeysi vasitəsilə təyin olunur və adətən verilənlər bazası sorğularında şərtli filtr tətbiqi üçün istifadə olunur. Bu, xüsusilə dinamik sorğuların (məsələn, istifadəçi tərəfindən seçilən filtrə əsasən) qurulmasında faydalıdır.

### Specification-in Xüsusiyyətləri
- **Dinamik Sorğular**: Statik JPQL və ya SQL sorğuları əvəzinə şərtləri dinamik şəkildə tətbiq edir.
- **Təkrar İstifadə**: Bir specification müxtəlif sorğularda təkrar istifadə oluna bilər.
- **Kombinasiya**: Çoxsaylı specification-lər `and()`, `or()`, `not()` kimi metodlarla birləşdirilə bilər.
- **Tip Təhlükəsizliyi**: JPA Criteria API ilə inteqrasiya olunaraq tip təhlükəsiz sorğular təmin edir.
- **Spring Data JPA ilə Uyğunluq**: `JpaSpecificationExecutor` interfeysi ilə repository-lərdə istifadə olunur.

### Specification İnterfeysi
```java
package org.springframework.data.jpa.domain;

@FunctionalInterface
public interface Specification<T> {
    Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder builder);
    
    // Statik metodlar
    static <T> Specification<T> where(Specification<T> spec) { ... }
    default Specification<T> and(Specification<T> other) { ... }
    default Specification<T> or(Specification<T> other) { ... }
    default Specification<T> not() { ... }
}
```
- **Abstrakt Metod**: `toPredicate()` – JPA Criteria API-də şərt (predicate) qurur.
- **Statik/Defolt Metodlar**: Şərtləri birləşdirmək və ya inkar etmək üçün.

### Spring Boot-da İstifadə
Spring Boot-da `Specification` istifadə etmək üçün repository `JpaSpecificationExecutor` interfeysini implement etməlidir:
```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.JpaSpecificationExecutor;
public interface UserRepository extends JpaRepository<User, Long>, JpaSpecificationExecutor<User> {
}
```

### Nümunə: Sadə Specification
Aşağıda `User` sinfi üçün adə əsasən filtr tətbiq edən specification nümunəsi göstərilir:
```java
import org.springframework.data.jpa.domain.Specification;
import jakarta.persistence.criteria.*;

public class UserSpecifications {
    public static Specification<User> hasName(String name) {
        return (root, query, builder) -> 
            builder.equal(root.get("name"), name);
    }
}
```
**İstifadə**:
```java
@Autowired
private UserRepository userRepository;

public List<User> findUsersByName(String name) {
    return userRepository.findAll(UserSpecifications.hasName(name));
}
```

### Specification-lərin Birləşdirilməsi
```java
public static Specification<User> hasAgeGreaterThan(int age) {
    return (root, query, builder) -> 
        builder.greaterThan(root.get("age"), age);
}

public List<User> findUsersByNameAndAge(String name, int age) {
    return userRepository.findAll(
        Specification.where(hasName(name)).and(hasAgeGreaterThan(age))
    );
}
```

### Üstünlüklər
- **Dinamiklik**: İstifadəçi girişinə əsasən şərtləri dəyişmək mümkündür.
- **Təkrar İstifadə**: Eyni specification müxtəlif sorğularda istifadə oluna bilər.
- **Oxunaqlılıq**: Şərtlər məntiqi və oxunaqlı formada təşkil edilir.
- **Tip Təhlükəsizliyi**: Criteria API ilə səhvlər kompilyasiya zamanı aşkarlanır.

### Çətinliklər
- **Mürəkkəblik**: Böyük və mürəkkəb sorğular üçün kod həcmi artar.
- **Öyrənmə Əyrisi**: Criteria API-nin sintaksisi yeni başlayanlar üçün mürəkkəb ola bilər.
- **Performans**: Dinamik sorğular statik JPQL-dən daha çox resurs tələb edə bilər.

---

## 2. Criteria API Nədir?

**Criteria API** Java Persistence API (JPA)-nin bir hissəsidir və tip təhlükəsiz, proqramlaşdırıla bilən sorğular yaratmaq üçün istifadə olunur. `jakarta.persistence.criteria` paketində yerləşir və SQL sorğularını əl ilə yazmaq əvəzinə obyekt-əsaslı sorğu qurmağa imkan verir. Spring Data JPA-da `Specification` interfeysi Criteria API üzərində qurulub.

### Criteria API-nin Xüsusiyyətləri
- **Tip Təhlükəsizliyi**: String əsaslı sorğular (JPQL) əvəzinə obyekt-əsaslı sorğular qurur, beləliklə kompilyasiya zamanı səhvlər aşkarlanır.
- **Dinamiklik**: Şərtlər runtime zamanı qurula bilər.
- **Esneklik**: Mürəkkəb sorğular (join, subquery, aggregation) dəstəklənir.
- **Portativlik**: JPA providerindən (Hibernate, EclipseLink) asılı olmayaraq işləyir.

### Əsas Komponentlər
1. **`CriteriaBuilder`**: Sorğu şərtlərini və ifadələrini yaratmaq üçün fabriktir.
   - Metodlar: `equal()`, `like()`, `greaterThan()`, `and()`, `or()`, və s.
2. **`CriteriaQuery`**: Sorğunun strukturunu təyin edir (SELECT, FROM, WHERE).
3. **`Root`**: Sorğunun əsas entity-sini təmsil edir.
4. **`Predicate`**: Şərt ifadəsini təmsil edir.
5. **`Path`**: Entity-nin atributlarına daxil olmaq üçün istifadə olunur.

### Criteria API ilə Sadə Sorğu
```java
import jakarta.persistence.criteria.*;
import jakarta.persistence.EntityManager;
import org.springframework.beans.factory.annotation.Autowired;

@Component
public class UserCriteriaService {
    @Autowired
    private EntityManager entityManager;

    public List<User> findUsersByName(String name) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> root = query.from(User.class);
        query.select(root).where(cb.equal(root.get("name"), name));
        return entityManager.createQuery(query).getResultList();
    }
}
```

### Spring Boot-da Specification ilə Criteria API
`Specification` interfeysi Criteria API-ni abstraktlaşdırır və daha oxunaqlı formada istifadəni təmin edir:
```java
public class UserSpecifications {
    public static Specification<User> hasNameAndAge(String name, int age) {
        return (root, query, builder) -> {
            Predicate namePredicate = builder.equal(root.get("name"), name);
            Predicate agePredicate = builder.greaterThan(root.get("age"), age);
            return builder.and(namePredicate, agePredicate);
        };
    }
}
```

### Mürəkkəb Sorğu Nümunəsi (Join ilə)
Aşağıda `User` və `Order` entity-ləri arasında join ilə sorğu göstərilir:
```java
public static Specification<User> hasOrderWithAmount(double amount) {
    return (root, query, builder) -> {
        Join<User, Order> orders = root.join("orders");
        return builder.greaterThan(orders.get("amount"), amount);
    };
}
```

### Üstünlüklər
- **Tip Təhlükəsizliyi**: Səhv atribut adları kompilyasiya zamanı aşkarlanır.
- **Dinamiklik**: Şərtlər runtime zamanı qurula bilər.
- **Mürəkkəb Sorğular**: Join, group by, subquery kimi kompleks əməliyyatlar dəstəklənir.
- **Təkrar İstifadə**: `Specification` ilə şərtlər modullaşdırılır.

### Çətinliklər
- **Mürəkkəb Sintaksis**: Böyük sorğular üçün kod həcmi artır.
- **Performans**: Dinamik sorğular statik sorğulardan daha yavaş ola bilər.
- **Öyrənmə Əyrisi**: Criteria API-nin obyekt-əsaslı strukturu SQL-dən fərqlidir.

---

## 3. Specification və Criteria API Arasındakı Fərqlər

| Xüsusiyyət                | Specification                            | Criteria API                            |
|---------------------------|------------------------------------------|-----------------------------------------|
| **Təyinat**               | Spring Data JPA-da dinamik sorğular      | JPA-nın ümumi sorğu qurma API-si        |
| **İnterfeys**             | `org.springframework.data.jpa.domain.Specification` | `jakarta.persistence.criteria` paketində |
| **Abstrakt Metod**        | `toPredicate()`                          | Çoxsaylı metodlar (`CriteriaBuilder`, və s.) |
| **İstifadə Sahəsi**       | Spring Data JPA repository-ləri          | Həm JPA, həm Spring Data ilə            |
| **Oxunaqlılıq**           | Daha sadə və modullu                    | Daha mürəkkəb sintaksis                |
| **Təkrar İstifadə**       | Asan (and, or, not metodları)           | Manual təkrar istifadə                 |

**Qeyd**: `Specification` Criteria API-ni daxildə istifadə edir, lakin Spring Data JPA üçün daha yüksək səviyyəli abstraksiya təmin edir.

---

## 4. Praktiki Nümunə: Dinamik Bilet Rezervasiya Sistemi

Aşağıda Spring Boot-da `Specification` və Criteria API istifadə edərək dinamik bilet rezervasiya sistemi göstərilir. Sistem istifadəçilərin adlarına, yaşlarına və ya sifariş miqdarına əsasən filtr tətbiq edir.

### Entity-lər
```java
import jakarta.persistence.*;
import java.util.List;

@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private int age;
    @OneToMany(mappedBy = "user")
    private List<Order> orders;
    // getters, setters
}

@Entity
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private double amount;
    @ManyToOne
    private User user;
    // getters, setters
}
```

### Specification-lər
```java
import org.springframework.data.jpa.domain.Specification;
import jakarta.persistence.criteria.*;

public class UserSpecifications {
    public static Specification<User> hasName(String name) {
        return (root, query, builder) -> 
            name == null ? null : builder.equal(root.get("name"), name);
    }

    public static Specification<User> hasAgeGreaterThan(int age) {
        return (root, query, builder) -> 
            age < 0 ? null : builder.greaterThan(root.get("age"), age);
    }

    public static Specification<User> hasOrderWithAmountGreaterThan(double amount) {
        return (root, query, builder) -> {
            Join<User, Order> orders = root.join("orders");
            return builder.greaterThan(orders.get("amount"), amount);
        };
    }

    public static Specification<User> dynamicFilter(String name, int age, double amount) {
        return Specification.where(hasName(name))
                            .and(hasAgeGreaterThan(age))
                            .and(hasOrderWithAmountGreaterThan(amount));
    }
}
```

### Repository
```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.JpaSpecificationExecutor;

public interface UserRepository extends JpaRepository<User, Long>, JpaSpecificationExecutor<User> {
}
```

### Service
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;

    public List<User> findUsers(String name, int age, double amount) {
        return userRepository.findAll(UserSpecifications.dynamicFilter(name, age, amount));
    }
}
```

### Controller
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import java.util.List;

@RestController
public class UserController {
    @Autowired
    private UserService userService;

    @GetMapping("/users")
    public List<User> getUsers(
        @RequestParam(required = false) String name,
        @RequestParam(required = false, defaultValue = "-1") int age,
        @RequestParam(required = false, defaultValue = "0.0") double amount
    ) {
        return userService.findUsers(name, age, amount);
    }
}
```

### İzahat
- **Dinamik Filtrlər**: `name`, `age`, və `amount` parametrlərinə əsasən istifadəçilər filtirlənir.
- **Tip Təhlükəsizliyi**: Criteria API ilə səhv atribut adları kompilyasiya zamanı aşkarlanır.
- **Join**: `User` və `Order` arasında əlaqə qurularaq sifariş məbləğinə əsasən filtr tətbiq edilir.
- **Modullu Structure**: `Specification` ilə şərtlər təkrar istifadə oluna bilər.

---

## 5. Specification və Criteria API ilə Problemlər və Həllər

### 5.1. Mürəkkəb Sorğular
**Problem**: Çoxsaylı join və ya subquery ilə sorğular yazmaq mürəkkəb ola bilər.
**Həll**:
- `Specification` ilə şərtləri modullaşdırın.
- Mürəkkəb sorğular üçün ayrı-ayrı `Specification` sinifləri yaradın.
- JPQL və ya native SQL istifadə etməyi də nəzərdən keçirin.

### 5.2. Performans
**Problem**: Dinamik sorğular statik sorğulardan yavaş ola bilər.
**Həll**:
- Lazımsız join və şərtlərdən qaçın.
- Query caching (`@QueryHints`) istifadə edin.
- İndeksləşdirmə ilə verilənlər bazası performansını optimallaşdırın.

### 5.3. Oxunaqlılıq
**Problem**: Böyük specification-lər oxunaqlılığı azalda bilər.
**Həll**:
- Şərtləri kiçik, təkrar istifadə oluna bilən `Specification`-lərə bölün.
- Metod adlarını mənalı seçin (məsələn, `hasName`, `hasAgeGreaterThan`).

---

## 6. Intervyu Sualları və Cavablar

### Sual 1: Specification ilə Criteria API arasındakı fərq nədir?
**Cavab**: `Specification` Spring Data JPA-da Criteria API-ni abstraktlaşdıran interfeysdir. `toPredicate()` metodu ilə şərtlər qurur və modullu, təkrar istifadə oluna bilən sorğular təmin edir. Criteria API isə JPA-nın birbaşa sorğu qurma mexanizmidir, daha aşağı səviyyəli və mürəkkəbdir.

### Sual 2: Specification nə üçün istifadə olunur?
**Cavab**: Dinamik sorğular yaratmaq, şərtləri təkrar istifadə etmək və tip təhlükəsizliyini təmin etmək üçün. Məsələn, istifadəçi filtrinə əsasən verilənlər bazasından dinamik axtarışlar üçün.

### Sual 3: Criteria API-nin üstünlükləri nələrdir?
**Cavab**: Tip təhlükəsizliyi, dinamiklik, mürəkkəb sorğular (join, subquery) dəstəyi və JPA providerindən asılı olmamaq.

### Sual 4: Specification ilə JPQL arasındakı fərq nədir?
**Cavab**: `Specification` tip təhlükəsizdir və dinamik sorğular üçün obyekt-əsaslıdır, lakin mürəkkəb ola bilər. JPQL string-əsaslıdır, daha sadədir, lakin kompilyasiya zamanı səhvlər aşkarlanmır.

### Sual 5: Dinamik sorğular üçün hansı hallarda Specification istifadə etmək məqsədəuyğundur?
**Cavab**: İstifadəçi girişinə əsasən dəyişən şərtlər, təkrar istifadə oluna bilən filtr mexanizmləri və tip təhlükəsizliyi tələb olunduqda.

---

## 7. Tövsiyələr
- **Specification**:
  - Şərtləri modullaşdıraraq təkrar istifadəni maksimum edin.
  - `and()`, `or()`, `not()` metodları ilə mürəkkəb şərtləri birləşdirin.
  - Null dəyərləri idarə etmək üçün şərt yoxlamalarını əlavə edin.
- **Criteria API**:
  - Mürəkkəb sorğular üçün `CriteriaBuilder`-in metodlarını tam öyrənin.
  - Join və subquery-lərdə performans optimizasiyasına diqqət edin.
  - Metamodel (JPA Static Metamodel Generator) ilə tip təhlükəsizliyini artırın.
- **Spring Boot-da**:
  - `JpaSpecificationExecutor` ilə dinamik sorğuları repository-lərdə tətbiq edin.
  - REST API-lərdə istifadəçi parametrlərinə əsasən `Specification` istifadə edin.
- **Intervyuya Hazırlıq**:
  - `Specification` və Criteria API-nin dinamik sorğulardakı üstünlüklərini izah edin.
  - Praktiki nümunələrlə (join, filtr) istifadə ssenarilərini nümayiş etdirin.
  - JPQL ilə müqayisədə tip təhlükəsizliyini vurğulayın.

---

## 8. Mənbələr
- Spring Data JPA Documentation: https://docs.spring.io/spring-data/jpa/docs/3.2.0/reference/html/
- Jakarta Persistence API: https://jakarta.ee/specifications/persistence/3.1/
- Baeldung on Specification: https://www.baeldung.com/spring-data-criteria-queries
- GeeksforGeeks on Criteria API: https://www.geeksforgeeks.org/jpa-criteria-api-introduction/

Bu sənəd Java və Spring Boot-da `Specification` və `Criteria API` haqqında ətraflı məlumat təqdim edir. Əlavə suallarınız varsa, əlaqə saxlayın!
