---
title: "Express에서 Clean Architecture 적용하기"
datePublished: Sun Feb 18 2024 11:28:28 GMT+0000 (Coordinated Universal Time)
cuid: clsrfcctj000108jp42jb0rvh
slug: express-clean-architecture
tags: express, nodejs, typescript

---

## Intro.

몇 개월 전에 솔루션 개발에 필요한 리서치를 끝내고, 솔루션 제품군 중에 하나를 도맡아 개발하기 시작했다.

최종적으로 내가 맡은 제품은 블록체인 노드와 함께 배포되어 동기/비동기 트랜잭션을 처리하고 블록체인 네트워크를 제어하는 역할을 하며, 타제품을 통해 들어오는 HTTP 요청이나 이벤트 메시지를 처리함과 동시에 새 이벤트를 생성해 전달하는 기능을 담당한다.

애플리케이션 규모가 크지 않고 이른 시일 내에 MVP를 개발해야 하는 점, 분산 환경에서 동기/비동기, 이벤트 등을 고려해야 한다는 점에서 Typescript, Node.js, 그리고 Express를 사용해 개발하기로 했다.

---

처음에는 규모가 크지 않다는 점에서 흔하게 볼 수 있는 Three Layered Architecture를 적용하려고 했다. 비즈니스 로직을 처리하는 Domain도 사실상 전혀 없고 Web3 라이브러리로 모든 프로세스를 처리했기 때문이다.

그러나 내가 만든 기능이 타입을 명확히 검증하고 있는지, 비동기로 동작하는지, 새 이벤트를 제대로 생성하고 있는지 등을 스몰 테스트로 확인하려 하는데 대부분 기능이 Web3 라이브러리와 블록체인 인프라에 의존하고 있어서 테스트하기 쉽지 않았다.

그래서 Clean Architecture를 적용해 라이브러리와 인프라에 의존하는 부분은 추상화와 DIP를 활용해 테스트 가능한 코드로 과감히 바꾸는 시도를 했다.

---

## Clean Architecture.

![Clean Coder Blog](https://blog.cleancoder.com/uncle-bob/images/2012-08-13-the-clean-architecture/CleanArchitecture.jpg align="left")

Clean Architecture는 소프트웨어 구조를 설계하는 개념 중 하나로, Onion Architecture라고도 한다.

이 아키텍처를 프로젝트에 적용했을 때 기대효과는 다음과 같다.

* 유지 보수성
    
* 테스트 가능성
    

---

Clean Architecture가 유지 보수성과 테스트 가능성에서 좋은 효과를 주는 이유는 이 아키텍처의 핵심 원칙인 **의존성 규칙**에 있다.

의존성 규칙이란 위 그림에서 볼 수 있듯이 각 계층이 독립적인 비즈니스 룰을 갖고 **외부 계층에서 내부 계층으로만 의존성이 흐르도록 하는 것**이다.

---

## Maintainable and Testable Code.

Clean Architecture는 유지 보수와 테스트 가능한 코드를 제공하게 한다. 그렇다면 어떻게 Clean Architecture가 그런 코드를 제공할 수 있는 것일까?

* **의존성 역전 원칙**
    
* **계층 분리**
    

첫째, **의존성 역전 원칙**을 적용한다. 이는 고수준 모듈이 저수준 모듈에 의존하지 않고 **추상화**에 의존하게 만드는 것이다. 위 그림(오른쪽 아래)에서 볼 수 있듯이, `Controller`는 Flow of Control에 따라 구현체가 아닌 `Use Case Input Port`라고 하는 추상화에 의존하고 있는 것을 확인할 수 있다. 또한 `Interactor`는 `Presenter` 구현체가 아닌 `Use Case Output Port`에 의존한다.

이러한 원칙은 테스트 할 때 저수준 모듈(실제 구현체)이 아닌 Mock, Stub이나 Fake 등으로 대체해 외부 의존성(데이터베이스, 다른 외부 서비스 등)으로부터 애플리케이션 로직을 독립시키기 때문에 필요한 설정 작업을 줄여주고, 테스트 실행 속도를 빠르게 한다.

---

```typescript
export interface UserRepository { 
    findById(id: string): Promise<User>
}

export class UserPgRepository implements UserRepository { 
    async findById(id: string): Promise<User> {
        // Postgres에서 User를 조회하는 로직이 있다고 가정 
        return user 
    }
}

export interface UserReadUsecase { 
    findById(id: string): Promise<User> 
}

export class UserReadService implements UserReadUsecase { 
    constructor(private readonly repository: UserReadRepository) {} 
    
    async findById(id: string): Promise<User> {
        return this.repository.findById(id)
    }
}
```

위 예제 코드와 같이 `UserReadService`는 의존성 역전 원칙을 통해 `UserPgRepository` 같이 특정 데이터베이스에 의존하는 구현체가 아닌 `UserRepository`라는 추상화에 의존하도록 한다.

이는 외부 인프라에 대한 의존성을 낮춰 테스트하기 쉽게 한다. 또한 코드에 대한 결합도를 낮춰 어떤 데이터베이스로 교체한다고 해도 `UserReadService`가 `UserRepository`에 의존하는 것을 변경할 필요가 없이 확장할 수 있다.

---

```typescript
class MockUserRepository implements UserRepository { 
    async findById(id: string): Promise<User> { 
        return { 
            id: id,
            name: 'Kyulog',
            email: 'kyulog@hashnode.dev'
        }
    }
}

describe('UserReadService', () => {
    it('should return a user by id', async () => {
        const fakeUserRepository = new MockUserRepository()
        const userReadService = new UserReadService(fakeUserRepository) 
        const user = await userReadService.findById(id)
        
        expect(user.id).toBe(id)
        expect(user.name).toBe('Kyulog')
        expect(user.email).toBe('kyulog@hashnode.dev')
    })
})
```

위 테스트 코드와 같이 개발자는 `UserReadService`가 실제 데이터베이스에 액세스하는 로직과 관련 없이 `UserRepository`를 Mock으로 구현해 테스트하는 데 사용할 수 있다. 이를 통해 원하는 기능을 단순하게 검증할 수 있고, 외부 의존성에 대한 설정과 네트워크 연결 요청이 없어서 테스트 속도가 빨라진다.

---

둘째, **계층 분리**를 적용한다. 이는 각 계층이 비즈니스 룰이라는 명확한 **책임과 역할**을 갖고 독립적인 계층으로 구성됨을 의미한다. 이렇게 함으로써 각 계층을 독립적으로 개발, 테스트, 유지보수할 수 있게 된다. 쉽게 말해, 코드를 수정하거나 기능을 추가할 때 다른 부분에 미치는 영향을 최소화할 수 있다는 것이다.

Clean Architecture는 다음과 같은 계층으로 분리될 수 있다.

* **엔티티**: 엔터프라이즈 비즈니스 룰을 캡슐화하는 객체로 데이터 구조나 핵심 비즈니스 로직과 정책이 정의된다. 다른 계층에서 이를 사용하기 때문에, 엔티티가 잘 정의되면 이 부분만 변경하거나 추가하여 쉽게 애플리케이션을 교체할 수 있다.
    
* **유즈케이스**: 애플리케이션 비즈니스 룰을 정의한다. 엔티티를 사용해 실제로 애플리케이션이 무엇을 해야 하는지를 일련의 프로세스로 실행하게 한다. 예를 들어, '사용자'가 '상품'을 구매하려 할 때, '상품' 데이터를 사용해 일련의 '구매 과정'을 진행하는 로직을 진행하는 것이다.
    
* **인터페이스 어댑터**: 웹 페이지, 모바일 앱 화면 등과 애플리케이션 사이에서 변환기 역할을 한다. 외부 요청을 애플리케이션이 필요한 형태로 변환하거나 비즈니스 로직의 결과를 외부에 전달할 수 있는 형태로 변환한다. 예를 들어, 웹 페이지에서 정보를 입력하면 그것을 적절하게 변환해 내부 로직에서 사용하게 한다.
    
* **프레임워크 & 드라이버**: 가장 바깥쪽에 있는 실제 데이터베이스, 웹 프레임워크, 디바이스 드라이버 등 외부 요소와 연결, 통신을 담당한다. 예를 들어, 데이터베이스에 정보를 저장하거나 다른 서비스의 API를 호출하는 것이 해당한다.
    

---

## Problem

Express 환경에서 Clean Architecture를 도입할 때 고려했던 사항 중에 유지 보수와 테스트 가능성 외에도 하나 더 있었다.

* 어떤 방법으로 의존성을 주입해야 하는가?
    

나는 원래 커리어 시작을 Spring Boot를 베이스로 시작했기 때문에 컨테이너가 제공해 주는 제어의 역전과 의존성 주입의 편리함에 취해있었다.

```java
@Service
@RequiredArgsConstructor
class UserService { 
    private final UserRepository userRepository; 
    
    public User create(String id, String password) { 
        User user = User.create(id, password);
        User user = userRepository.save(user);
        return user; 
    } 
}
```

Spring Boot로 개발을 해봤다면 위와 같은 형태의 코드 구조에 익숙할 것이다.

따라서 `UserRepository`는 컨테이너에 Bean으로 등록했기 때문에 제어의 역전으로 컨테이너가 주입해주는 원리라는 것, 그래서 개발자는 인스턴스를 직접 생성해 줄 필요가 없다는 것을 알고 있을 것이다.

이런 과정이 내겐 너무나 익숙하고 당연한다 생각했기 때문에 안일한 생각으로 Express 환경에서도 자연스럽게 그런 코드를 만들어 냈는데, Express는 Spring Boot에 견줄 만한 컨테이너 기술을 가지고 있지 않기 때문에 역시나 코드 곳곳에서 빨간 줄을 볼 수 있었다.

* Express는 IoC/DI를 해 줄 컨테이너가 없다.
    

Express와 함께 사용되는 컨테이너 라이브러리에는 대표적으로 `inversify`, `typedi`, `tsyringe` 등이 있다. 이 라이브러리의 도움으로 컨테이너를 만들어 의존성을 관리할 수 있지만, 그렇게까지 할 필요는 없다고 생각했다. 그리고 컨테이너 코드를 개발자가 직접 관리해야 한다는 단점도 있었다.

---

## Try.

그렇다면 이 상황에서 나는 어떻게 Express 환경에서 의존성을 주입하고 테스트 가능한 코드를 만들어 냈을까?

먼저 객체지향 프로그래밍 패러다임에 기반해 생각해 본다면 Class를 사용해 볼 수 있었다.

```typescript
// user-create-controller.ts
export class UserCreateController { 
    constructor(private readonly userCreateService: UserCreateUsecase) {}
    async execute(req: Request, res: Response): Promise<void> {
        const data: UserCreate = req.body
        const response = UserCreateResponse.from(await this.userCreateService.execute(data))
        res.status(200).json(response)
    }
}

// user-create-usecase.ts 
export class UserCreateService implements UserCreateUsecase { 
    constructor(private readonly userRepository: UserRepository) {} 
    async execute(data: UserCreate): Promise<User> { 
        const user = User.create(...data)
        const newUser = await this.userRepository.save(user)
        return newUser 
    }
}
```

Class로 작성하게 된다면 위와 같은 코드를 작성할 수 있다. 큰 이슈는 없어 보인다. 그러나 이렇게 작성되면 보기 싫은 패턴이 하나 생긴다.

```typescript
// dependency-container.ts 
const userRepository = new UserRepository()
const userCreateService = new UserCreateService(userRepository) 
export const userCreateController = new UserCreateController(userCreateService)

// user-router.ts
export const UserRouter = Router()
UserRouter.post('/api/users/user', userCreateController.execute)
```

위와 같이 `dependency-container.ts`라는 모듈을 하나 생성해서 의존성을 따로 관리하는 패턴이 만들어질 것이다. 이는 내가 앞서 말했던 컨테이너 라이브러리를 사용했을 때 개발자가 컨테이너 코드를 직접 관리해야 하는 문제와 일치한다. 위 코드는 결국 규모가 커져 컴포넌트가 늘어날수록 추가해야 할 코드가 늘어나 관리하기 복잡해질 수 있다고 생각했다.

---

Class를 활용했을 때의 문제점 때문에 나는 각 계층의 로직들을 함수형으로 작성하기로 했다. 또한 이들을 역할과 책임을 적절히 분리해 로직을 작성해 갔다.

코드의 구조화, 상태 관리와 객체의 불변성을 제공해야 하는 부분은 Class로 작성했다.

```typescript
// user-repository.ts 
export const UserPgRepository: UserRepository = { 
    create: async (user: User): Promise<User> => { 
        const query = 'INSERT INTO users(id, name, email) VALUES($1, $2, $3) RETURNING *'
        const values = [user.id, user.name, user.email]

        const { rows } = await pg.query(query, values) 
        const newUser: User = UserPgEntity.from(rows[0]) 
        return newUser 
    }
}

// user-create-controller.ts
export const UserCreateController = (userCreateService: UserCreateUsecase) => {
    return async (req: Request, res: Response) => {
        const data: UserCreate = req.body
        const user = await userCreateService.execute(data)
        res.status(200).json(user)
    }
}

// user-create-usecase.ts 
export const UserCreateService = (userRepository: UserRepository): UserCreateUsecase => ({ 
    execute: async (data: UserCreate): Promise<User> => {
        const user = User.create(...data)
        const newUser = await userRepository.create(user)
        return newUser 
    }
})

// user-router.ts
export const UserRouter = Router() 
UserRouter.post('/api/users/user', UserCreateController(UserCreateService(UserRepository)))
```

---

## Conclusion.

이렇게 함수형으로 Clean Architecture를 적용할 수 있다. 그리고 이를 테스트 하는 방법은 Class를 활용했을 때와 같다.

위에서 `UserReadService`를 테스트하는 코드를 빌려와 아래와 같이 변경할 수 있다.

```typescript
const MockUserRepository: UserRepository = { 
    findById: async (id: string): Promise<User> => { 
        return { 
            id: id,
            name: 'Kyulog',
            email: 'kyulog@hashnode.dev'
        }
    }
}

describe('UserReadService', () => {
    it('should return a user by id', async () => {
        const userReadService = UserReadService(MockUserRepository) 
        const user = await userReadService.findById(id)
        
        expect(user.id).toBe(id)
        expect(user.name).toBe('Kyulog')
        expect(user.email).toBe('kyulog@hashnode.dev')
    })
})
```

함수형으로 작성해도 외부 의존성 없이 테스트 가능한 코드를 작성할 수 있다는 것이 확인된다.

---

그러나 클로저를 사용하는 것이 문제가 될 소지가 있다고 생각될 수 있다. 하지만 이는 Express가 구조적으로 유지해야 하는 부분이기 때문에 어쩔 수 없는 부분이기도 하며, Express의 요청 처리 모델이 각 요청에 대해서 독립적인 실행 컨텍스트를 가지고 있고, 요청 처리가 완료된 후 해당 컨텍스트와 관련된 모든 데이터와 클로저는 **도달 불가능한 상태**가 되기 때문에 GC에 의해 메모리에서 해제되어 큰 문제가 없다.

실제로 함수형으로 작성했을 때와 Class로 작성했을 때 부하 테스트를 통한 메모리 사용량을 검사해 보았다.

`Artillery`를 사용해 60초간 100명의 가상 사용자가 회원 가입한다고 했을 때, Class로 작성된 Express 애플리케이션의 경우, 모든 테스트를 마쳤을 때 최종 메모리 사용량은 **101.421875MB**였으며, 함수형으로 작성된 코드의 경우에는 **101.234375MB**로 큰 차이가 없었다.

> 만약에 메모리 사용량을 줄이고 싶다면 비동기 I/O 작업을 우선 검사하고 최적화 방안을 찾는 게 포인트일 것 같다.

이 외에 데이터 처리량과 전송량에 대해서도 검사해 보았다.

* 처리량의 경우에는 함수형으로 작성된 코드가 최초 10 times에 **828 count**, Class로 작성된 코드가 최초 10 times에 **682 count**로 함수형으로 작성된 코드가 더 많은 요청을 처리했다.
    
* 전송량의 경우에는 함수형으로 작성된 코드가 최초 10 times에 **30,636 bytes**, Class로 작성된 코드가 최초 10 times에 **25,234 bytes**로 Class로 작성된 코드가 더 적게(가볍게) 전송했다.
    

이러한 결과는 자바스크립트에서 함수와 클로저, 그리고 Class 인스턴스를 처리하는 방식에 차이가 있기 때문일 수 있다. 그러나 이 둘은 테스트 시간이 경과 하면서 비슷한 수치를 보여 주었고, 이 같은 단일 지표만으로 어떤 것의 성능이 더 우수하다고 판단하기에는 어렵다고 생각한다.

---

## End.

찾아보니까 나처럼 OOP와 Functional을 적절히 조합해 개발하고 있는 분들이 몇몇 계신거 같다.

* [https://dev.to/bespoyasov/clean-architecture-on-frontend-4311](https://dev.to/bespoyasov/clean-architecture-on-frontend-4311)
    
* [https://blog.bitsrc.io/a-clean-and-adaptive-nodejs-architecture-with-typescript-b144c1735447](https://blog.bitsrc.io/a-clean-and-adaptive-nodejs-architecture-with-typescript-b144c1735447)
    

아무튼 은탄환은 없다. 각자 프로젝트 규모와 개발팀 스타일에 맞게 개발하도록 하자.

> "No Silver Bullet" - Fred Brooks, 1986, ⟪**No Silver Bullet – Essence and Accident in Software Engineering**⟫

---

## Future Work.

* 일부 코드에 함수형을 채택 하면서 발생할 수 있는 메모리 누수를 검사할 수 있어야 한다.
    
* 또한, 내가 작성한 코드에 CPU-Intensive한 코드가 있을 수 있기 때문에, 이를 해결해 보려 한다.