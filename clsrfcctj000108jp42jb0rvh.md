---
title: "Express에서 Clean Architecture 적용하기"
datePublished: Sun Feb 18 2024 11:28:28 GMT+0000 (Coordinated Universal Time)
cuid: clsrfcctj000108jp42jb0rvh
slug: express-clean-architecture
tags: express, nodejs, typescript

---

## Intro.

몇 개월 전에 솔루션 개발에 필요한 리서치를 끝내고, 솔루션 제품군 중에 하나를 도맡아 개발하기 시작했다.

최종적으로 내가 맡은 제품은 블록체인 노드와 함께 배포되어 동기/비동기 트랜잭션을 처리하고 블록체인 네트워크를 제어하는 역할을 하며, 타제품을 통해 들어오는 HTTP 요청이나 이벤트 메시지를 처리함과 동시에 새 이벤트를 생성해 전달하는 기능을 담당한다.

애플리케이션 규모가 크지 않고 빠른 시일 내에 MVP를 개발해야 하는 점, 분산 환경에서 동기/비동기, 이벤트 등을 고려해야 한다는 점에서 Typescript, Node.js, 그리고 Express를 사용해 개발하기로 했다.

---

처음에는 규모가 크지 않다는 점에서 흔하게 볼 수 있는 Three Layered Architecture를 적용하려고 했다. 비즈니스 로직을 처리하는 Domain도 사실상 전무하고 Web3 라이브러리로 모든 프로세스를 처리했기 때문이다.

그러나 내가 만든 기능이 타입을 명확히 검증하고 있는지, 비동기로 동작하는지, 새 이벤트를 제대로 생성하고 있는지 등을 스몰 테스트로 확인하려 하는데 대부분의 기능이 Web3 라이브러리와 블록체인 인프라에 의존하고 있어서 테스트하기 쉽지 않았다.

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

이러한 원칙은 테스트 할 때 저수준 모듈(실제 구현체)이 아닌 Mock이나 Stub으로 대체해 외부 의존성(데이터베이스, 다른 외부 서비스 등)으로 부터 애플리케이션 로직을 독립시키기 때문에 필요한 설정 작업을 줄여주고, 테스트 실행 속도를 빠르게 한다.

## Problem #2.

Express 환경에서 Clean Architecture를 도입할 때 고려했던 사항 중에 테스트 가능성 외에도 하나 더 있었다. 어떤 방법으로 의존성을 주입해야 하는가였다.

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

따라서 `UserRepository` 는 컨테이너에 Bean으로 등록했기 때문에 제어의 역전으로 컨테이너가 주입해주는 원리라는 것, 그래서 개발자는 인스턴스를 직접 생성해 줄 필요가 없다는 것을 알고 있을 것이다.

이런 과정이 내겐 너무나 익숙하고 당연한 것이라 생각했기 때문에 안일한 생각으로 Express 환경에서도 자연스럽게 그런 코드를 만들어 냈는데, Express는 Spring Boot에 견줄 만한 컨테이너 기술을 가지고 있지 않기 때문에 역시나 코드 곳곳에서 빨간 줄을 볼 수 있었다.

> Express와 함께 사용되는 컨테이너 라이브러리에는 대표적으로 `inversify`, `typedi`, `tsyringe` 등이 있다. 이 라이브러리의 도움으로 컨테이너를 만들어 의존성을 관리할 수 있지만, 그렇게 까지 할 필요는 없다고 생각했다. 그리고 컨테이너 코드를 개발자가 직접 관리해야 한다는 단점도 있었다.

---

## Try.

그렇다면 나는 어떻게 Express에서 의존성을 주입했고, 테스트 가능한 코드를 만들어 냈을까?

먼저 객체지향 프로그래밍 패러다임에 기반해 생각해 본다면 Class를 사용해 볼 수 있을 것이다.

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

Class로 작성하게 된다면 위와 같은 코드를 작성할 수 있다. 큰 이슈는 없어보인다.

그러나 이렇게 작성되면 보기 싫은 패턴이 하나 생긴다.

```typescript
// dependency-container.ts 
const userRepository = new UserRepository()
const userCreateService = new UserCreateService(userRepository) 
export const userCreateController = new UserCreateController(userCreateService)

// user-router.ts
export const UserRouter = Router()
UserRouter.post('/api/users/user', userCreateController.execute)
```

위와 같이 `dependency-container.ts` 라는 모듈을 하나 생성해서 의존성을 따로 관리하는 패턴이 만들어질 것이다. 이는 내가 앞서 말했던 컨테이너 라이브러리를 사용했을 때 개발자가 컨테이너 코드를 직접 관리해야 하는 문제와 일치한다.

또한, 정의 시점과 인스턴스화 시점이 동일하고 인스턴스가 전역에서 관리되기 때문에 애플리케이션이 실행되고 런타임에 메모리가 계속 참조된 채로 남아있게 된다.

---

## Conclusion.

Class를 활용했을 때의 문제점 때문에 나는 컨트롤러나 서비스 같은 것들은 함수형으로 작성하면서 모듈 패턴을 적극 이용하기로 했다.

```typescript
// user-create-controller.ts
export const UserCreateController = (userCreateService: UserCreateUsecase) => {
    return { 
        execute: async (req: Request, res: Response) => { 
            const data: UserCreate = req.body
            const response = UserCreateResponse.from(await userCreateService.execute(data))
            res.status(200).json(response)
        }
    }
}

// user-create-usecase.ts 
export const UserCreateService = (userRepository: UserRepository): UserCreateUsecase => { 
    return { 
        execute: async (data: UserCreate): Promise<User> => {
            const user = User.create(...data)
            const newUser = await userRepository.save(user)
            return newUser 
        }
    }
}

// user-router.ts
export const UserRouter = Router() 
UserRouter.post('/api/users/user', UserCreateController(UserCreateService(UserRepository)).execute)
```

이렇게 되면 의존성을 관리하는 코드를 따로 작성하지 않아도 된다. 동시에 Class의 정의 시점과 인스턴스화 시점이 동일해 메모리 해제가 되지 않던 문제를 해결하게 된다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1708438066219/42ab43d6-d5c7-4867-8b46-4839b4bd1cde.png align="center")

실제로 함수형을 쓴 것(`adaptor`)과 그렇지 않은(`admin`), 비슷한 규모의 애플리케이션을 비교했을 때 메모리 사용량은 사진과 같이 몇 배 가량 차이가 난다.

> "No Silver Bullet" - Fred Brooks, 1986, ⟪**No Silver Bullet – Essence and Accident in Software Engineering**⟫

그렇다고 해서 함수형이 더 좋다고 할 순 없다. 함수형을 사용하게 되면 상태 관리가 어렵다. 상태 관리를 해야 하면 불변성을 유지하기도 어렵다. 또한, 함수 중첩과 클로저로 복잡한 규모의 애플리케이션에서는 메모리 관리가 어려울 수 있다.

위 코드에서 `UserRouter`가 애플리케이션의 생명주기 동안에 `UserCreateController`와 `UserCreateService`를 참조하고 있기 때문에 클로저들은 메모리에서 해제되지 않을 것이다.

> `UserCreateController` 입장에서 캡처되는 `userCreateService`, `UserCreateService` 입장에서 캡처되는 `userRepository` 는 요청에 대한 처리가 끝나면 메모리 해제된다.

그러나 이는 Express가 구조적으로 유지해야 하는 부분이기 때문에 어쩔 수 없는 것 같다. 각자 프로젝트 규모와 개발팀 스타일에 맞게 개발하도록 하자.

---

## Future Work.

* 일부 코드에 함수형을 채택 하면서 발생할 수 있는 메모리 누수를 검사할 수 있어야 한다.
    
* 또한, 내가 작성한 코드에 CPU-Intensive한 코드가 있을 수 있기 때문에, 이를 해결해 보려 한다.