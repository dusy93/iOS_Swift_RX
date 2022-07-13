- **Just** : **오직 하나만** 출력하고 완료 처리 ( 배열을 넣으면 배열 통째로 한번 출력하고 종료 )

```swift
let disposeBag = DisposeBag()

Observable.just("Hi")
	.subscribe(onNext: { print($0) })
  	.disposed(by: disposeBag)

Observable.just([1, 2, 3, 4, 5], scheduler: MainScheduler.instance)
            .subscribe(onNext: { print($0) })
            .disposed(by: disposeBag)

// ***출력*** 
// Hi
// [1, 2, 3, 4, 5]
```

- **Of** : 배열을 넣으면 배열로 출력하고 완료 처리 ( **하나의 인스턴스를 여러개 넣을 때 사용**하면 좋을듯 )

```swift
let disposeBag = DisposeBag()

let one = "One"
let two = "Two"
let three = "Three"

Observable.of(one, two, three)
            .subscribe(onNext: { print($0, terminator: " ") })
            .disposed(by: disposeBag)

// ***출력*** 
// one two three
```

- **From** : 배열을 넣으면 **배열안에 있는 값을 하나씩 출력**하고 완료 처리 ( 배열에 사용하면 좋을듯 )

```swift
let disposeBag = DisposeBag()

let nameArray = ["yeon", "ging", "ju"]

Observable.from(nameArray)
            .map{ $0 + "😉" }
            .subscribe(onNext: { print($0) })
            .disposed(by: disposeBag)

// ***출력*** 
// yeon😉
// ging😉
// ju😉
```

- **withUnretained**
    - self를 따로 선언해주지 않아도 사용할 수 있음

 - 원래 사용하던 코드

```swift
let disposeBag = DisposeBag()

viewModel.setting
	.subscribe(onNext: { [weak self] info in
		guard let self = self else { return }
		self.setInfo(with: info)
	})
	.disposed(on: disposeBag)
```

 - withUnretained 사용한 코드

```swift
let disposeBag = DisposeBag()

viewModel.setting
	.withUnretained(self)
	.subscribe(onNext: { this, info in
		this.setInfo(with: info)
	})
	.disposed(on: disposeBag)
```

- **withLatestFrom**
    - 첫번째 observable 항목이 방출될 때마다 두번째 observable의 가장 최근 값이 함께 방출

```swift
model.settings
	.withLatestFrom(model.oldSettings) { ($0, $1) }
	.map { new, old in
		print("new : \(new) | old : \(old) ")
		return new.name
	}
	.bind(to: model.name)
	.disposed(by: disposeBag)

// model.settings의 값이 새로 들어오면 model.oldSettings의 가장 최근 값도 같이 가져감
```

- **DistinctUntilChanged**
    - 연달아 중복된 값이 들어올 경우 무시

```swift
textFiled.rx.text
	.distinctUntilChanged()

// 또는

model.select
	.distinctUntilChanged()
	.bind(to: model.selectIndex)
	.disposed(by: disposeBag)
```

- **distinctUntilChanged :** 중복된 값이 오면 이후 값은 무시

```swift
let disposeBag = DisposeBag()

let numArray = [1, 1, 2, 2, 2, 3, 3, 4, 5, 5, 5, 5, 5]

Observavble.from(numArray)
	.distinctUnilChanged()
	.subscribe(onNext: { print($0) })
	.disposed(by: disposeBag)

// ***출력*** 
// 1
// 2
// 3
// 4
// 5
```

- **subscribe**
    - 대상의 상태가 변한 값을 받아 처리
    - 다음 값, 에러, 완료 타이밍을 알 수 있음

- **bind**
    - 단순히 새로 생성되는 값을 넘겨줄 때 간편하게 사용
    - 에러 컨트롤 불가

- **drive**
    - bind 와 비슷하지만 main scheduler(main thread)에서 동작
    - UI와 관련된 코드를 작성할 때 쓰이는 편

```swift
let disposeBag = DisposeBag()

viewModel.name
	.asDriver()
	.drive(onNext: { [weak self] in
		guard let self = self else { return }
		self.event = true
	})
	.disposed(by: disposeBag)

// 위의 코드를 drive가 아닌 subscribe으로 사용했을 때

viewModel.name
	.observeOn(MainScheduler.instance)
	.subscribe(onNext: { [weak self] in
		guard let self = self else { return }
		self.event = true
	})
	.disposed(by: disposeBag)
```

> **asDriver** : Subject나 relay를 drive 값을 할당해주기 위해 달아준다
> 
> 
> ```swift
> // model
> **var** type = BehaviorRelay<Int>(value: 0)
> 
> // viewModel
> type = model.type
> 	.asDriver(onErrorRecover: {_ in .empty()})
> 
> // viewController+Bind
> viewModel.accessType
> 		.drive{ [weak self] type in
>			guard let self = self else{ return }
> 			...
> 		}
> 		.disposed(by: bag)
> ```
> 

- **Publish(subject/relay)** : 빈 상태로 시작해 새로운 값만 내보냄
    - 처음 연결한 후 이후에 들어오는 값을 받음

- **Behavior(subject/relay)** : 초기값을 가진 상태로 시작하여 최신값을 내보냄
    - 새로 들어온 값이 없는 경우 초기값 내보냄
    - 연결하자마자 이전에 있던 최신값을 받고 시작함

- **Replay(subject/relay)** : buffer 사이즈 만큼 최신 값을 내보냄
    - buffer 크기만큼 채워져야 내보냄

> **Subject** vs **Relay**
Subject : .completed , .error 이벤트가 발생하면 subscribe가 종료됨
Relay :  .completed , .error 이벤트를 발생하지 않고 Dispose되기 전까지 계속 작동,
             UI Event에 사용하기 좋음
> 

- **CombineLatest**
    - A, B 두개의 observable을 전달 받아 하나의 값을 받았을 때 마지막 값이 있으면 합쳐서 내보냄
    - 최대 8개까지 observable 가능

<aside>
✔️ A : 1———2——3———
B : —a—b—————c—
출력 : 1a—1b—2b—3b—3c

</aside>

```swift
let disposeBag = DisposeBag()

let AObservable = PublishSubject<Int>()
let BObservable = PublishSubject<Int>()

Observable
	.combineLatest(AObservable, BObservable){ ($0,$1) }
	.map{ a, b in 
		print("a : \(a) | b : \(b)")
		return a+b
	}
	.bind(to: model.sum)
	.disposed(by: disposeBag)		
```

- **Zip**
    - 짝을 맞춰서 전달(중복 요소는 전달되지 않음)

<aside>
✔️ A : 1———2——3———
B : —a—b—————c—
출력 : 1a—2b—3c

</aside>

```swift
let disposeBag = DisposeBag()

let AObservable = PublishSubject<Int>()
let BObservable = PublishSubject<String>()

Observable
	.zip(AObservable, BObservable) { ($0, $1) }
	.map{ a, b in
		print("a : \(a) | b : \(b)")
		return "\(a) : \(b)"
	}
	.bind(to: model.number)
	.disposed(by: disposeBag)
```

- **Concat**
    - 내부 요소들을 순차적으로 연결하여 하나의 observable로 반환

<aside>
✔️ A : 1———2——3———
B : —a—b—————c—
출력 : 1—2—3—a—b—c

</aside>

```swift
let disposeBag = DisposeBag()

let aArray = Observable.from([1, 2, 3, 4, 5])
let bArray = Observable.from([6, 7, 8, 9, 10])

Observable
	.concat([aArray, bArray])
	.subscribe(onNext: { print ($0) })
	.disposed(by: disposeBag)
```

- **Merge**
    - 내부 요소들을 연결하여 하나의 observable로 반환, concat처럼 순서가 보장되지 않음

<aside>
✔️ A : 1———2——3———
B : —a—b—————c—
출력 : 1—a—b—2—3—c

</aside>

```swift
let disposeBag = DisposeBag()

let aArray = Observable.from([1, 2, 3, 4, 5])
let bArray = Observable.from([6, 7, 8, 9, 10])

Observable
	.merge([aArray, bArray])
	.subscribe(onNext: { print ($0) })
	.disposed(by: disposeBag)
```

- **Map**
    - observable 값을 변형하여 내보냄

```swift
let numbers = [1, 2, 3]
let transformNum = numbers
			.map{ $0 * 2 }
			.map{ $0 + 1 } // 여러개의 map 이어서 사용 가능

// ***출력*** 
// 3
// 5
// 7
```

- **FlatMap**
    - n차원 배열을 n-1차원 배열로 변경됨

```swift
let numList = [["one", 2, 3], [4, "five", 6]]
let numMap = numList.flatMap { Int($0) }

// ***출력*** 
// [nil, 2, 3, 4, nil, 6]
```

- **CompactMap**
    - nil이 아닌 결과값을 출력

```swift
let numList = [["one", 2, 3], [4, "five", 6]]
let numMap = numList.compactMap { Int($0) }

// ***출력*** 
// [[2, 3], [4, 6]]
```

> **Map** vs **FlatMap** vs **CompactMap**
map : 단순히 값을 다른 값으로 변환하는 경우
flatMap : 결과를 한 차원 아래로 평면화하는 경우
compactMap : nil 값을 제거해야하는 경우
> 

- **Throttle**
    - 입력 시 바로 값이 보내지고 특정 시간동안 대기
    - 버튼 중복 입력 방지에 사용

```swift
btn.rx.tap
	.asDriver()
	.throttle(.seconds(2))
	.drive(onNext: { (_) in
		self.btnEvent()
	})
	.disposed(by: disposeBag)
```

<aside>
✔️ .throttle(.seconds(2), **latest: Bool**)

*- latest가 true인 경우*
btn          : a—b—**c**—**d**—**e**—f—
seconds : |—**1**—||—2—||—3—|
event      : a—— b—— d—— f

*- latest가 false인 경우*
btn          : a—b—**c**—**d**—**e**—f—
seconds : |—**1**—||—2—||—3—|
event      : a——- c——- e——-

</aside>

- **Debounce**
    - 입력 시 특정 시간동안 대기 후 값이 보내짐
    - 텍스트 필드 입력에 사용

```swift
btn.rx.tap
	.asDriver()
	.debounce(.seconds(2))
	.drive(onNext: { (_) in
		self.btnEvent()
	})
	.disposed(by: disposeBag)
```

<aside>
✔️ btn          : a——b——**c**——**d**——**e**——f——
seconds : |——**1**——||——2——||——3——|
event      :  -———— b———— d————  f

</aside>

- **Filter**
    - 해당 조건에 만족하는 요소들만 이벤트로 전달

```swift
let disposeBag = DisposeBag()
let numbers = [1, 2, 3, 4, 5, 6, 7, 8]

Observable
	.from(numbers)
	.filter{ $0 % 3 = 0 }
	.subscribe(onNext: { print ($0) })
	.disposed(by: disposeBag)

// ***출력*** 
// 3
// 6
```

- **Take** : n개의 항목만 내보냄

```swift
let disposeBag = DisposeBag()

let numArray = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

Observable.from(numArray)
	.take(3)
	.subscribe(onNext: { print($0) })
	.disposed(by: disposeBag)

// ***출력*** 
// 1
// 2
// 3
```

- **TakeLast** : 가장 마지막부터 n개의 항목 내보냄
    - 위에 코드에서 .takeLast(2)로 했을 때 9, 10 출력

- **Skip**
    - 처음 발생하는 n개의 이벤트를 무시

```swift
let disposeBag = DisposeBag()

Observable.of("A", "B", "C", "D")
	.skip(2)
	.subscribe(onNext: { print($0) })
	.disposed(by: disposeBag)

// ***출력*** 
// C
// D
```

- **SkipWhile**
    - filter와 비슷하지만 filter는 조건이 true인 값이 전달되지만 skipwhile은 false인 값이 전달됨
    - 조건이 한번이라도 false면 그 다음은 계속 전달됨

```swift
Observable.of(2, 2, 1, 3, 4)
	.skip(while: { $0 % 2 == 0 })
	.subscribe(onNext: { print($0) })
	.disposed(by: bag)

// ***출력*** 
// 1
// 3
// 4
```

- **SkipUntil**
    - 다른 observable이 들어올때까지 전달되지 않다가 특정 observable이 들어온 순간부터 전달된

- **IgnoreElements**
    - .next 이벤트는 무시하여 항목을 내보내지 않지만 .error 나 .complete 전달(종료되는 시점 알 수 있음)

- **ElementAt**
    - 발생하는 이벤트 중 n번째 이벤트부터 받고싶을 때 사용
    - 인덱스는 0부터 시작

```swift
Observable.of(1, 2, 3, 4, 5)
      .element(at: 2)
      .subscribe(onNext: { print($0)})
      .disposed(by: bag)

// ***출력*** 
// 3
// 4
// 5
```
