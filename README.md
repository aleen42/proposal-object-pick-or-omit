## `Object.{pick,pickBy}` / `Object.{omit,omitBy}`

ECMAScript Proposal, specs, and reference implementation for `Object.pick`, `Object.pickBy`, `Object.omit`, and `Object.omitBy`.

Spec drafted by [@Aleen](https://github.com/aleen42).

### Motivation

- To operate an object convenient by picking or omitting its properties, described in [the group](https://es.discourse.group/t/object-prototype-pick-object-prototype-omit/515).

### Syntax

```
Object.pick(obj[, pickedKeys])
Object.pickBy(obj[, predictedFunction(currentValue[, key[, object]])[, thisArg]])

Object.omit(obj[, omittedKeys])
Object.omitBy(obj[, predictedFunction(currentValue[, key[, object]])[, thisArg]])
```

#### Parameters

- `obj`: which object you want to pick or omit.
- `pickedKeys` (**optional**): keys of properties you want to pick from the object. The default value is empty array.
- `omittedKeys` (**optional**): keys of properties you want to pick from the object. The default value is empty array.
- `predictedFunction` (**optional**): the function to predicted whether the property should be picked or omitted. The default value is an identity: `x => x`.
  - `currentValue`: the current value processed in the object.
  - `key`: the key of the `currentValue` in the object.
  - `object`: the object `pickBy` or `omitBy` was called upon.
- `thisArg` (**optional**): the object used as `this` inside the predicted function.

#### Returns

Returns a new object, which has picked or omitted properties from the object.

### Usage

```js
// default
Object.pick({a : 1}); // => {}
Object.omit({a : 1}); // => {a: 1}
Object.pickBy({a : 0, b : 1}); // => {b: 1}
Object.omitBy({a : 0, b : 1}); // => {a: 0}
Object.pickBy({}, function () { console.log(this) }); // => the object itself
Object.pickBy({}, function () { console.log(this) }, window); // => Window

Object.pick({a : 1, b : 2}, ['a']); // => {a: 1}
Object.omit({a : 1, b : 2}, ['b']); // => {b: 1}

Object.pick({a : 1, b : 2}, ['c']); // => {}
Object.omit({a : 1, b : 2}, ['c']); // => {a: 1, b: 2}

Object.pick([], [Symbol.iterator]); // => {Symbol(Symbol.iterator): f}
Object.pick([], ['length']); // => {length: 0}

Object.pickBy({a : 1, b : 2}, v => v === 1); // => {a: 1}
Object.omitBy({a : 1, b : 2}, v => v === 2); // => {a: 1}
Object.pickBy({a : 1, b : 2}, (v, k) => k === 'a'); // => {a: 1}
Object.omitBy({a : 1, b : 2}, (v, k) => k === 'b'); // => {a: 1}
```

### FAQ

1. When it comes to the prototype chain of an object, should the method pick or omit it? (The answer may change)

    A: The implementation of [`_.pick`](https://lodash.com/docs/4.17.15#pick) and [`_.omit`](https://lodash.com/docs/4.17.15#omit) by Lodash has taken care about the chain. To keep the rule, we can pick off properties of prototype, but can't omit them:

    ```js
    Object.pick({a : 1}, ['toString']); // => {toString: f}
    Object.omit({a : 1}, ['toString']).toString; // => ƒ toString() { [native code] }
    ```

    The same rule applies to the `__proto__`:

    ```js
    Object.pick({}, ['__proto__']); // => {__proto__: {...}}
    Object.omit({}, ['__proto__']).__proto__; // => {...} 
    ```

2. What is the type of the returned value?

    A: All these methods should return plain objects:

    ```js
    Object.pick([]); // => {} 
    Object.omit([]); // => {}
    Object.pick(new Map()); // => {} 
    Object.omit(new Map()); // => {}
    ```

3. How to handle `Symbol`?

    A: `Symbol` should just be considered as properties within `Symbol` keys, and they should obey the rules mentioned above:

    ```js
    Object.pick([], [Symbol.iterator]); // => {Symbol(Symbol.iterator): f}, pick off from the prototype
    Object.omit([], [Symbol.iterator]); // => {}, plain objects
    
    const symbol = Symbol('key');
    Object.omit({a : 1, [symbol]: 2}, [symbol]); // => {a : 1}
    
    Object.prototype[symbol] = 'test'; // override prototype
    Object.pick({}, [symbol]); // => {Symbol(key): "test"}, pick off from the prototype
    Object.omit({}, [symbol])[symbol]; // => "test", cannot omit properties from the prototype
    ```

4. If some properties of an object is not accessible like throwing an error, can `Object.pick` or `Object.omit` operate such an object?

    A: I suggest throwing the error wrapped by `Object.pick` or `Object.omit`, but it is **NOT the final choice**:

    ```js
    Object.pick(Object.defineProperty({}, 'key', {
	    get() { throw new Error() }
    }), ['key']);
    ```

    The error stack will look like:

    ```
    Uncaught Error
        at Object.get (<anonymous>:2:20)
        at Object.pick (<anonymous>:2:10)
        at <anonymous>:1:8 
    ```

5. In comparison with [**proposal-shorthand-improvements**](https://github.com/rbuckton/proposal-shorthand-improvements), when should we use these two methods?

    A: Multiple properties. Assume that we need to ensure an object without any side-effected keys except `key1` and `key2`:

    ```js
    postData({[key1] : o[key1], [key2] : o[key2]});
    postData(Object.pick(o, [key1, key2])); 
    ```

6. Why can't be defined on the `Object.prototype` directly?

    A: As `Object` is especially fundamental, and both of them will result in conflicts of properties of any other objects. In shorthand, if defined, any objects inherited from `Object` with `pick` or `omit` defined in its prototype should break.

