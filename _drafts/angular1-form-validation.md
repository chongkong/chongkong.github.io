---
layout: post
title: AngularJS Form Validation
date: 2016-10-23 07:07:41 +0900
categories: angular1
tags: ['frontend', 'angular', 'angular1', '삽질']
---

요 며칠간 AngularJS form validation 처리를 하느라 고생을 좀 했습니다. 알고보면 별 것 아닌데, form validation 내용들을 
잘 정리해 둔 글은 별로 없어서 한 번 정리해 보려고 합니다.

## Basics

Form validation에 참여하는 주체들이 뭐가 있고, 각각이 어떤 역할을 하는지 이해하는 것이 가장 중요합니다. 설명을 위해서 다음과 같은 간단한
디렉티브 구조를 생각해 봅시다.

``` html
<div ng-form name="myForm">
    <input type="number" name="age" ng-model="user.age">
</div>
```

### NgModelController

가장 먼저 `ng-model` 디렉티브를 통해 생성되는 [`NgModelController`](https://docs.angularjs.org/api/ng/type/ngModel.NgModelController)를 알아봅시다.
`NgModelController`는 데이터 바인딩 등을 담당하는데, Form Validation에서 가장 중요한 역할인 [`$modelValue`](https://docs.angularjs.org/api/ng/type/ngModel.NgModelController#$modelValue)
의 validation을 `NgModelController`가 수행하게 됩니다.
구체적으로는 다음과 같습니다.

- `NgModelController`는 `$modelValue`의 validition 로직을 관리합니다.
- [`$validators`](https://docs.angularjs.org/api/ng/type/ngModel.NgModelController#$validators)에는 
string key마다 정의된 validator 함수가 붙어있습니다.

``` javascript
// ctrl은 NgModelController의 인스턴스를 의미합니다
ctrl = {
    $modelValue: 3,
    $validators: {
        // "min" validator
        min: function(modelValue) { return modelValue > 5; }
    }
}
```

- `$modelValue`가 변경되면 `$validators`에 있는 모든 validator 함수들이 실행됩니다. validation 결과값이 `false`인 validator key들은
  `$error` 객체에 들어갑니다. 단 하나의 validation 결과라도 `false`라면 validation에는 실패했다고 간주됩니다. `$modelValue`가 변경되지 않아도
  [`$validate()`](https://docs.angularjs.org/api/ng/type/ngModel.NgModelController#$validate) 함수를 실행하면 같은 동작이 일어납니다. 

``` javascript
ctrl = {
    $modelValue: 3,
    $valid: false,
    $invalid: true,
    $validators: {
        min: function(modelValue) { return modelValue > 5; }
    },
    $error: {
        min: ["¯\_(ツ)_/¯"]  // validation of "min" validator failed
    }
}
```

- `$validators`나 `$validate()` 외에도 `$setValidity(key, value)` 함수를 통해서 `$error` 객체 내부에 key를 생성할 수 있습니다.

``` javascript
ctrl.$setValidity('awesome', false);
ctrl = {
    $valid: false,  // $error 객체가 비어있지 않으므로 invalid
    $invalid: true,
    $error: {
        awesome: ["¯\_(ツ)_/¯"]
    }
}
```

### FormController



## $validators를 이용한 Custom Form Validation

## $setValidity를 이용한 Custom Form Validation

## Upgrade

### Custom CSS

### Validation Class
