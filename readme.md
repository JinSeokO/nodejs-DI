# Node.js에서의 DI

![resources/Node.js_logo.svg](resources/Node.js_logo.svg)

# Over View

- Node.js를 이용하여 DI(Dependency Injection)에 대한 이해를 도와주기 위해 본 문서를 작성한다.

# Design Goal

- Dependency의 기본 개념 이해
- DI에 대한 이해 및 DI의 장/단점 인식
- Ioc의 개념 이해

# 관계

개발에서 관계란 클래스와 클래스 간의 관계를 일반적으로 일컬으며 일반적으로 크게 아래와 같이
4가지로 분류된다.

- Generalization(일반화)
- Realization(실체화)
- Dependency(의존)
- Association(연관)

이중에서 DI와 연관이 있는 Dependency를 간단한 domain 소스를 이용해서 설명하려고 한다.

Dependency(의존성) 이란 간단하게 설명하자면 클래스 또는 함수(A)가 사용하기 위해서
다른 객체(B)를 참조하는 것을 의미한다.

즉 A를 사용하기 위해서는 B가 필요하며 의존하고 있다고 볼 수 있다.

간단한 Dependency의 예를 알아보자. 

```jsx
/// cartRepository.js
export const newCartRepository = () => {
	async function save(item) {
		... // logic
	}
	async function getAll() {
		... // logic
	}
}
```

```jsx
// cartService.js
import { newCartRository } from '../repository/cartRepository'

export const newCartService = () => {
	async function addItem (item) {
	... // domain logic
  const cartItemId = await cartRepository.save(item);
  return cartItemId;
	}

	async function getAll () {
	  return await cartRepository.findAll();
	}

	return {
	  addItem,
		getAll,
	}
}
```

Cart service는 cart repository를 변수 생성 및 사용함에 따라 각 function에서 의존함에 따라 dependency, 즉 의존 관계로 볼 수 있다.

다만, 위 예제는 변수로 선언하여 사용함에 따라,

강한 결합(Tight coupling)으로서 높은 의존성을 가지고 있다.

또한,  위 예제는 SRP(Single Responsibility Principle / 단일 책임 원칙)을 위반한 사례로도 볼 수 있다.

Service 에서는 service 로직에 대해서만 책임을 가져야 하나 cartRepository 생성에 대한 로직도

참조하고 있음에 따라 SRP를 위반 사례라고 볼 수도 있다.

해당 사례로 인해 발생하는 문제점에 대한 예를 들면 test를 들 수 있다.

```jsx
import { newCartService } from 'cartService'

describe('test cart service', () => {
	test('test add cart item', async () => {
		const cartService = newCartService();
		const item = {
	    name: 'test item'
    };
		
		// 여기서 의존중인 repository에 대한 mockup을 해주어야하나
		// 필드로 선언이 됨에 따라 해당 cartRepository는 mockup이 불가능
		// 테스트를 위해서는 별도 mockup library가 필요
		// 또는 repository를 직접사용
		const insertedCartItemId = await cartService.addItem(item);
	}
});
```

위와 같이 테스트 시 cartRepository는 cartService가 생성될 시에 내부 필드로 초기화 됨에 따라

domain로직 만을 테스트하기가 어려워진다.

또한, cart의 종류가 여러가지일 때, cartRepository에 따라 맞는 cartService 를 새로 코드를 짜 주어야 한다는 단점이 있다.

# DI(Dependency Injection / 의존성 주입)

DI는 위와 같이 Tight Coupling(강한 결합)을  Loose Coupling(느슨한 결합)으로 전환 시키는 방법이다.

DI는 내부 필드로 선언 및 초기화 하는 것이 아닌 Constructor, Function, Field 에 주입 함으로서
다음과 같이 여러가지 이점을 가지게 된다.

- Testability를 향상시킨다
- 변경에 유연한 코드를 가진다
- 의존하는 객체가 interface일 경우 여러 객체를 사용할 수 있다.

## DI의 종류

DI에는 총 3가지의 방법이 존재하며 다음과 같다

1. Field Injection

    ```jsx
    // cartService.js
    export const newCartService = () => {
    	let cartRepository;

    	async function addItem (item) {
    	  ... // domain logic
      	  const cartItemId = await cartRepository.save(item);
      	  return cartItemId;
    	}

    	async function getAll () {
    	  return await cartRepository.findAll();
    	}

    	return {
    	  cartRepository
    	  addItem,
    	  getAll,
    	}
    }

    // app.js
    import { newCartService } from 'cartService'
    import { newCartRository } from '../repository/cartRepository'

    const main = async () => {
    	const cartService = newCartService();
    	cartService.cartRepository = newCartRepository();
    	console.log(await cartService.getAll());
    }

    main();
    ```

2. Function(Setter) Injection

    ```jsx
    // cartService.js
    export const newCartService = () => {
    	let cartRepository;

    	function setCartRepository (newCartRepository) {
    	  cartRepository = newCartRepository;
    	}

    	async function addItem (item) {
    	  ... // domain logic
          const cartItemId = await cartRepository.save(item);
          return cartItemId;
    	}

    	async function getAll () {
    	  return await cartRepository.findAll();
    	}

    	return {
    	  setCartRepository,
    	  addItem,
    	  getAll,
    	}
    }

    // app.js
    import { newCartService } from 'cartService'
    import { newCartRository } from '../repository/cartRepository'

    const main = async () => {
    	const cartService = newCartService();
    	cartService.setCartRepository(newCartRepository());
    	console.log(await cartService.getAll());
    }

    main();
    ```

3. Constructor Injection

    ```jsx
    // cartService.js
    export const newCartService = ({ cartRepository }) => {
    	if (!cartRepository) {
    		Throw new Error('cartRepository is empty');
    	}

    	async function addItem (item) {
    	  ... // domain logic
          const cartItemId = await cartRepository.save(item);
          return cartItemId;
    	}

    	async function getAll () {
    	  return await cartRepository.findAll();
    	}

    	return {
    	  cartRepository
    	  addItem,
    	  getAll,
    	}
    }

    // app.js
    import { newCartService } from 'cartService'
    import { newCartRository } from '../repository/cartRepository'

    const main = async () => {
    	const cartRepository = newCartRepository();
    	const cartService = newCartService({cartRepository});
    	console.log(await cartService.getAll());
    }

    main();
    ```

이 중에서 Constructor Inject(생성자 주입)이 많은 Design pattern에서 권장된다.

권장되는 이유는 다음과 같다.

1. **필수적으로 사용하여야 하는 의존성 없이는 해당 instance를 생성할 수 없도록 강제 할 수 있다.**

    Field Injection과 Function(Setter) Injection의 경우 의존성 없이 생성을 할 수 있게 됨에 따라

    에러가 발생할 가능성이 높아진다.

2. **불변성을 지킬 수 있다.**

    Field Injection과 Function(Setter) Injection의 경우 cartRepository를 변경 할 수 있게 됨에 따라
    해당 cartService의 상태의 불변성이 깨지게 된다.

3. **의존하는 컴포넌트가 많을 경우 생성자의 인자가 많아져 위기감을 가질 수 있게된다.**

    의존하는 컴포넌트가 많을 경우 일반적으로 SRP(Single Responsibility Principle)를 위반하는
    객체임에 따라 refactoring을 할 수 있게 기회를 제공한다

## DI를 이용한 TEST

DI를 이용할 경우 test시 별도 mocking library를 사용할 필요 없이 자체 mockup을 만들 수 있다.

```jsx
import { newCartService } from 'cartService'

describe('test cart service', () => {
  test('test add cart item', async () => {
    const mockCartRepository = () => {
      return {
        getAll: () => {
	  return [
	    ... // 객체
	  ]
	},
	save: (item) => {
	  return 1;
	}
      }
    }
    const cartService = newCartService({cartRepository: mockCartRepository});
    const item = {
      name: 'test item'
    };
    const expectCartItemId = 1;

    const insertedCartItemId = await cartService.addItem(item);
    expect(insertedCartItemId).toBe(expectCartItemId);
  }
});
```

# IoC(Inversion of Control / 제어의 역전)

일반적으로 개발자가 프로그램의 흐름을 제어하는 주체였다면, IoC의 개념이 나오게 되며
프레임워크가 dependency를 container화(java에서는 bean)시켜 생명주기를 관리게 되었다.

즉, dependency의 제어권이 개발자에서 프레임워크로 넘어가게 되었으며 이를 제어권의 흐름이

변경되었다고 하여 IoC(Inversion of Control)이라고 하게된다.

대표적인 프레임워크로는 Java Spring / Typescript ts-di가 있다.

# 참고문헌

1. [의존성 주입], 위키백과([https://ko.wikipedia.org/wiki/의존성_주입](https://ko.wikipedia.org/wiki/%EC%9D%98%EC%A1%B4%EC%84%B1_%EC%A3%BC%EC%9E%85))
2. [Inversion of Control vs Dependency Injection], stackoverflow([https://stackoverflow.com/questions/6550700/inversion-of-control-vs-dependency-injection](https://stackoverflow.com/questions/6550700/inversion-of-control-vs-dependency-injection))
