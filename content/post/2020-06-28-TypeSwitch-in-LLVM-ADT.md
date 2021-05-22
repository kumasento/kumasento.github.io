---
layout: post
title: TypeSwitch in LLVM ADT
author: Ruizhe Zhao
comments: true
---

This post summarizes what I've learned after viewing the [source code of `TypeSwitch`](https://llvm.org/doxygen/TypeSwitch_8h_source.html), which I think is neatly implemented and has much to learn.`

# Usage

```cpp
Operation *op = ...;
LogicalResult result = TypeSwitch<Operation *, LogicalResult>(op)
  .Case<ConstantOp>([](ConstantOp op) { ... })
  .Default([](Operation *op) { ... });
```

- `TypeSwitch` has two template arguments: `T` the type of the input, and `ResultT`, the type of the result from the `switch` statement;
- Each `Case` tries to match the input element to the template type of that `Case`. If matched, call the lambda function;
- There is a `Default` statement to capture all other cases.

# Implementation

`TypeSwitch` makes use of the C++ RTTI feature. It's basic idea is that, for each `Case` to be matched, we use `dyn_cast` (from LLVM) to make a typecast attempt; if failed, we move on to the next `Case`.

## Class Hierarchy

## Virtual Base Class

There is a virtual base class `TypeSwitchBase<Derived, T>`, which follows CRTP. It can be constructed by a **parameterized** constructor and a **move** constructor. The copy constructor has been removed.

### `Case`

`TypeSwitchBase` has a base implementation of `Case`. See the snippet below.

```cpp
template <typename CaseT, typename CaseT2, typename... CaseTs,
          typename CallableT>
DerivedT &Case(CallableT &&caseFn) {
  DerivedT &derived = static_cast<DerivedT &>(*this);
  return derived.template Case<CaseT>(caseFn)
      .template Case<CaseT2, CaseTs...>(caseFn);
}
```

- It accepts a template parameter pack `CaseTs` and three other argument placeholders.
- It takes in an rvalue reference `caseFn`, which is normally a *temporary* lambda function.
- It runs by first getting the *reference* to the typecast `DerivedT` self (`*this`). The reason for getting a reference is because of the return type, and the possible use case by chaining multiple `Case` together (see [this](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern#Polymorphic_chaining)).
- The stuff it returns is a recursively `Case` chain. All these `Case` share the same `caseFn`.

Another `Case` implementation by `TypeSwitchBase` is like this:

```cpp
template <typename CallableT> DerivedT &Case(CallableT &&caseFn) {
	using Traits = function_traits<std::decay_t<CallableT>>;
	using CaseT = std::remove_cv_t<std::remove_pointer_t<
	   std::remove_reference_t<typename Traits::template arg_t<0>>>>;
	
	DerivedT &derived = static_cast<DerivedT &>(*this);
	return derived.template Case<CaseT>(std::forward<CallableT>(caseFn));
}
```

It features the case when only one template argument is available, and that argument is the type of the `caseFn`. Its purpose is to infer the type of `Case` from he first input of the given `CallableT` object. 

To understand how this works, we should first get familiar with several LLVM and `std` utilities.

- `function_traits` ([ref](https://llvm.org/doxygen/structllvm_1_1function__traits.html)): it provides various traits information about a callable object. Specifically, you can access to the *type* of the arguments of that callable object.
- `std::decay_t` ([ref](https://en.cppreference.com/w/cpp/types/decay)): it first removes reference of `CallableT &&` and then converts `CallableT` from function to pointer. Therefore, `std::decay_t<CallableT>` gives a function pointer to `CallableT`.
- `std::remove_cv_t`, `std::remove_pointer_t`, `std::remove_reference_t` removes specific qualifiers (const volatile, pointer, and reference) of types. Their overall effect updates the type of the first argument of `caseFn` we can get from `function_traits`.
- `std::forward` ([ref](https://en.cppreference.com/w/cpp/utility/forward)): needed to deal with lvalue and rvalue conversion.

Based on these information, we can realize how this function deduces the type of the argument from `CallableT`, and forward the function call to `Case<CaseT>`, in which `CaseT` is the deduced type.

### `castValue`

This is the function that makes the attempt to perform typecast. There are two different implementation, which are distinguished based on whether the input `value` has `dyn_cast` member function provided or not. They will return `nullptr` if the typecast attempt is not successful.

```cpp
template <typename CastT, typename ValueT>
static auto castValue(
   ValueT value,
   typename std::enable_if_t<
       is_detected<has_dyn_cast_t, ValueT, CastT>::value> * = nullptr) {
 return value.template dyn_cast<CastT>();
}

template <typename CastT, typename ValueT>
static auto castValue(
   ValueT value,
   typename std::enable_if_t<
       !is_detected<has_dyn_cast_t, ValueT, CastT>::value> * = nullptr) {
 return dyn_cast<CastT>(value);
}
```

## TypeSwitch

`TypeSwitch<T, ResultT>` is a template class: `T` is the type of input, and `ResultT` is the type of the switch result.

There are two `TypeSwitch` variants inherited from `TypeSwitchBase`, although they are specializing template arguments differently.

For the variant with returned result, it has a `Optional` field `result` in it, which will be `null` after initialization. Its `Case` is implemented as follows:

```cpp
template <typename CaseT, typename CallableT>
TypeSwitch<T, ResultT> &Case(CallableT &&caseFn) {
 if (result)
   return *this;

 // Check to see if CaseT applies to 'value'.
 if (auto caseValue = BaseT::template castValue<CaseT>(this->value))
   result = caseFn(caseValue);
 return *this;
}
```

It has a rather simple logic: if `result` is assigned, `Case` immediately returns; otherwise, it tries to cast the input value to the corresponding `CaseT`, and assign the returned value to `result`. In this way, there will be one instance in the chain of `Case` that successfully assign `result` a not null value, and that value will stay there.

The other variant without returned value implements `Case` in a similar way, only that the tracked `result` member is replaced by `foundMatch`, a boolean that indicates whether there is a successful typecast in the chain of `Case`. 

One thing to note is that when calling `caseFn` in `Case`, the input argument is already typecast.

# Example usage

Here is a code snippet from the MLIR [tutorial](https://github.com/llvm/llvm-project/blob/master/mlir/examples/toy/Ch1/parser/AST.cpp). This `ASTDumper` intends to dump the content of an expression in a textual format. `ExprAST` is the base class to all the classes in the template arguments of `Case`. It simply tries to dispatch the `dump` function call to a subclass specific implementation. The lambda function here captures input parameters by reference. 

```cpp
/// Dispatch to a generic expressions to the appropriate subclass using RTTI
void ASTDumper::dump(ExprAST *expr) {
  llvm::TypeSwitch<ExprAST *>(expr)
      .Case<BinaryExprAST, CallExprAST, LiteralExprAST, NumberExprAST,
            PrintExprAST, ReturnExprAST, VarDeclExprAST, VariableExprAST>(
          [&](auto *node) { this->dump(node); })
      .Default([&](ExprAST *) {
        // No match, fallback to a generic message
        INDENT();
        llvm::errs() << "<unknown Expr, kind " << expr->getKind() << ">\n";
      });
}
```