---
title: "Javascript 배열 고차 함수에 대해 알아보자"
datePublished: Sat Mar 23 2024 11:30:07 GMT+0000 (Coordinated Universal Time)
cuid: clu40dfwv000309l64hjj48vv
slug: javascript-array-higher-order-function
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/i9gR_dz_xzU/upload/8dd5239170810aab0ae3945a154ea2a0.jpeg
tags: javascript, array, higher-order-functions

---

자바스크립트에서 배열은 다양하게 사용됩니다. 이 때 고차함수와 함께 이용할 수 있는 다양한 함수들이 있습니다. 자바스크립트 배열에서 어떤 함수들이 고차함수를 사용할 수 있는지 간략하게 정리하고, 심화 학습을 해보도록 하겠습니다.

> 고차함수: Higher Order Function 속칭 HOF 라는 별칭으로도 불리우는데요, 함수를 임자로 전달받거나 함수를 결과로 반환하는 함수를 말합니다.

## 고차 함수를 사용하는 배열 함수들

1. map: 배열의 각 요소에 대해 주어진 고차 함수를 실행하고 결과를 모아서 새 배열을 만듭니다.
    
    ```javascript
    const nums = [1, 2, 3, 4, 5]; 
    const squared = nums.map(num => num * num); // [1, 4, 9, 16, 25]
    ```
    
2. filter: 주어진 고차 함수에서 true를 반환하는 모든 요소를 모아 새 배열을 만듭니다.
    
    ```javascript
    const nums = [1, 2, 3, 4, 5]; 
    const evens = nums.filter(num => num % 2 === 0); // [2, 4]
    ```
    
3. reduce: 배열의 각 요소에 대해 주어진 고차 함수를 실행하고 하나의 결과값을 만듭니다. (생략: 비슷한 함수인데 역순으로 순회하는 reduceRight도 있습니다
    
    ```javascript
    const nums = [1, 2, 3, 4, 5]; 
    const sum = nums.reduce((acc, curr) => acc + curr, 0); // 15
    ```
    
4. forEach: 배열의 각 요소에 대해 주어진 고차 함수를 실행합니다.
    
    ```javascript
    const nums = [1, 2, 3, 4, 5]; 
    nums.forEach(num => console.log(num));
    ```
    
5. find: 주어진 고차 함수를 통해 판별 조건을 만족하는 첫 번째 요소를 반환합니다. 없으면 `undefined`를 반환합니다.
    
    ```javascript
    const nums = [5, 12, 8, 130, 44];
    const found = numbers.find(num => num > 10); // 12
    ```
    
6. some: 배열의 어떤 요소라도 주어진 함수를 만족하는지 확인합니다. true가 하나라도 있으면 true를 반환합니다.
    
    ```javascript
    const nums = [5, 12, 8, 130, 44];
    const hasLargeNum = nums.some(num => num > 100); // true
    ```
    
7. every: `some`과 유사하지만, 배열의 모든 요소가 주어진 함수를 만족하는지 확인합니다. false가 하나라도 있으면 false를 반환합니다.
    
    ```javascript
    const nums = [5, 12, 8, 130, 44];
    const allPositive = nums.every(num => num > 0); // true
    ```
    
8. sort: 배열의 요소들을 정렬합니다.
    
    ```javascript
    const nums = [5, 12, 8, 130, 44];
    const sortedNums = nums.sort((a, b) => a - b); // [5, 8, 12, 44, 130]
    ```
    
9. findIndex: 주어진 함수를 만족하는 배열의 첫 번째 요소의 인덱스를 반환합니다. 없으면 -1 반환합니다.
    
    ```javascript
    const nums = [5, 12, 8, 130, 44];
    const largeNumIndex = nums.findIndex(num => num > 10); // 1
    ```
    
10. flatMap: 배열의 각 요소에 대해 함수를 실행한 결과를 평평하게 펼친 새 배열을 만듭니다.
    
    ```javascript
    const nums = [5, 12, 8, 130, 44];
    const flatMappedNums = nums.flatMap(num => [num, num * 2]);
    // [5, 10, 12, 24, 8, 16, 130, 260, 44, 88]
    ```
    

## ES2022에서 추가된 배열 고차함수

1. findLast: 배열을 역순으로 순회하며 주어진 함수를 만족하는 첫 번째 요소의 값을 반환합니다.
    
    ```javascript
    const nums = [5, 12, 8, 130, 44, 12];
    const lastLargeNum = nums.findLast(num => num > 10); // 12
    ```
    
2. findLastIndex(): 배열을 역순으로 순회하며 주어진 함수를 만족하는 첫 번째 요소의 인덱스를 반환합니다.
    
    ```javascript
    const nums = [5, 12, 8, 130, 44, 12];
    const indexLastLargeNum = nums.findLastIndex(num => num > 10); // 5
    ```
    

## ES2023 에서 추가된 배열 고차함수

1. toReversed: reverse에 대응 되는 복사 메서드입니다. ([복사 메서드 설명](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Array#%EB%B3%B5%EC%82%AC_%EB%A9%94%EC%84%9C%EB%93%9C%EC%99%80_%EB%B3%80%EA%B2%BD_%EB%A9%94%EC%84%9C%EB%93%9Cmutating_method))
    
    ```javascript
    const nums = [1, 2, 3, 4, 5];
    const reversedNums = nums.toReversed(); //  [5, 4, 3, 2, 1]
    ```
    
2. toSorted: sort에 대응되는 복사 메서드입니다.
    
    ```javascript
    const nums = [5, 3, 4, 1, 2];
    const sortedNums = nums.toSorted((a, b) => a - b); // [1, 2, 3, 4, 5]
    ```
    
3. toSpliced: splice에 대응되는 복사 메서드입니다.
    
    ```javascript
    const nums = [1, 2, 3, 4, 5];
    const splicedNums = nums.toSpliced(1, 2, 6, 7); // [1, 6, 7, 4, 5]
    ```
    

> 한 가지 특이사항으로 ES2023의 복사 메서드들은 만약 배열 값 중 희소 배열 이 있을 경우 undefined로 대체하여 성능 저하 위협 요인을 방지합니다.

## 고차 함수 사용 시 주의사항

1. **성능 고려하기**: 고차 함수의 연쇄 사용시 큰 데이터 셋에서는 성능 저하를 일으킬 수 있습니다. 순회의 횟수가 많다면 고민이 필요합니다.
    
2. **순수 함수 사용하기**: 고차 함수에 전달되는 콜백 함수는 외부 상태를 변경하지 않는 동일 입력 동일 결과를 리턴하는 순수 함수를 이용하면 좋습니다.
    
3. `this`**값 주의하기**: `forEach`, `map`, `filter` 같은 고차 함수에서는 함수 내부의 `this`의 값이 예상과 다를 수 있습니다. 고차 함수의 두번째 인자로 `this`를 명시적으로 전달하거나, bind를 따로 하고 넘기던가, 화살표 함수를 사용하는 방식으로 대응이 필요합니다.
    
4. **원본 배열 불변성 유지**: 원본 배열을 조작하는 경우 예상치 못한 동작을 초래할 수 있습니다. 스프레드 연산자 등을 활용해서 원본 배열과의 연관을 끊는 등 방어적 코드를 작성해주면 좋습니다.
    

## React에서 활용 되는 고차 함수

* **컴포넌트 재사용성 높일 때 사용**: 고차 컴포넌트(Higher-Order Components, HOC) 구현시 활용되는 고차 함수.
    
    * 컴포넌트를 인자로 받아 새로운 컴포넌트를 반환합니다.
        
    * 중복 로직이나 관심사를 분리할 수 있습니다.
        
    * **예시: 로딩 상태 관리를 위한 HOC**
        
        ```javascript
        // withLoading.js
        function withLoading(Component) {
          return function WithLoadingComponent({ isLoading, ...props }) {
            if (isLoading) return <p>Loading...</p>;
            return <Component {...props} />;
          };
        }
        
        // 사용 예
        // ListComponent는 실제 컴포넌트
        const ListWithLoading = withLoading(ListComponent);
        
        // App.js에서 사용
        function App() {
          const [isLoading, setIsLoading] = useState(true);
          const [items, setItems] = useState([]);
        
          useEffect(() => {
            // fetchItems는 다른데 구현되있는 fetch 함수를 호출하는 의사 코드로 생각해주세요
            fetchItems().then(data => {
              setItems(data);
              setIsLoading(false);
            });
          }, []);
        
          return <ListWithLoading isLoading={isLoading} items={items} />;
        }
        ```
        
* **커스텀 훅에서의 활용**: 여러 컴포넌트에서 재사용할 수 있는 커스텀 훅에서 사용. 컴포넌트 로직을 모듈화하고 추상화합니다.
    
    * **예시) 데이터 fetching을 위한 커스텀 훅**
        
        ```javascript
        // useFetch.js
        function useFetch(url) {
          const [data, setData] = useState(null);
          const [isLoading, setIsLoading] = useState(true);
        
          useEffect(() => {
            fetch(url)
              .then(response => response.json())
              .then(data => {
                setData(data);
                setIsLoading(false);
              });
          }, [url]);
        
          return { data, isLoading };
        }
        
        // 컴포넌트에서 사용
        function DataComponent({ url }) {
          const { data, isLoading } = useFetch(url);
        
          if (isLoading) return <p>Loading...</p>;
          return <div>{JSON.stringify(data)}</div>;
        }
        ```
        

## 마치며

개인적으로 자바스크립트 배열에 대해 공부를 하며 고차 함수에 대해서 좀 더 정리해보고 싶어서 글을 작성 했습니다. 개인적으로 고차 함수 사용시의 주의사항에 대해 알아보며 선언적 프로그래밍의 패러다임을 따르려면 고차 함수의 사용에 너무 매몰 되는 것도 위험할 수 있다는 것을 다시 상기할 수 있었습니다.