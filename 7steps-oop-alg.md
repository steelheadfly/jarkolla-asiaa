# 7 steps from traditional OOP Objects to Algebraic Objects

### A convoluted example to start with

```
class Score {
  constructor(num) {
    if (typeof num == 'number' && !isNaN(num)) {
      if (num >= 0 && num <= 99) {
        this.num = num
      } else {
        throw new Error('Score not in valid range score range')
      }
    } else {
      throw new Error('Constructor for Score expects a number')
    }
  }
  inc() {
    if (this.num < 99) {
      this.num = this.num+1
    } else {
      throw new Error('Score not in valid range score range')
    }
  }
  dec() {
    if (this.num > 0) {
      this.num = this.num-1
    } else {
      throw new Error('Score not in valid range score range')
    }
  }
  double() {
    if (this.num * 2 <= 99) {
      this.num = this.num * 2
    } else {
      throw new Error('Score not in valid range score range')
    }
  }

}
```

### 1. Move generic arithmetic functions outside of the class

Replace all arithmetic methods with an update method that takes a normal function from n -> n to run inside the class context.

```
class Score {
  constructor(num) {
    if (typeof num == 'number' && !isNaN(num)) {
      if (num >= 0 && num <= 99) {
        this.num = num
      } else {
        throw new Error('Score not in valid range score range')
      }
    } else {
      throw new Error('Constructor for Score expects a number')
    }
  }
  update(f) {
    let num = this.num;
    if (typeof f(num) == 'number' && !isNaN(f(num))) {
      if (f(num) >= 0 && f(num) <= 99) {
        this.num = f(num)
      } else {
        throw new Error('Score not in valid range score range')
      }
    }
  }
}

let inc = n => n+1;
let dec = n => n-2;
let double = n => n*2

let myScore = new Score(12)

myScore.update(double);

console.log(
  myScore
)
```

### 2. Move type and range checking routine outside of the class

Move the type checkers and range validation that are useful in many other classes to external functions and pass them in to the class

```
class Score {
  constructor(num, predicate, constraint) {
    if (predicate(num)) {
      if (constraint(num)) {
        this.num = num;
        this.predicate = predicate;
        this.constraint = constraint;
      } else {
        throw new Error('Score not in valid range score range')
      }
    } else {
      throw new Error('Constructor for Score expects a number')
    }
  }
  update(f) {
    let { num, predicate, constraint } = this;
    if (predicate(f(num))) {
      if (constraint(f(num))) {
        this.num = f(num)
      } else {
        throw new Error('Score not in valid range score range')
      }
    }
  }
}

let isNumber = a => typeof a === 'number' && !isNaN(a);
let isInRange = n => n >= 0 && n <= 99;

let inc = n => n+1;
let dec = n => n-2;
let double = n => n*2

let myScore = new Score(51, isNumber, isInRange)

myScore.update(double);

console.log(
  myScore
)
```

### 3. Add an Error object within the Score Set

Return the new ScoreError instead of throwing. This way we can call update method on both objects in this set with valid returns for both cases.

```
class Score {
  constructor(num, predicate, constraint) {
    if (predicate(num)) {
      if (constraint(num)) {
        this.num = num;
        this.predicate = predicate;
        this.constraint = constraint;
      } else {
        return new ScoreError('Score not in valid range score range')
      }
    } else {
      return new ScoreError('Constructor for Score expects a number')
    }
  }
  update(f) {
    let { num, predicate, constraint } = this;
    if (predicate(f(num))) {
      if (constraint(f(num))) {
        this.num = f(num)
      } else {
        return new ScoreError('Score not in valid range score range')
      }
    }
  }
}

class ScoreError {
  constructor(error) {
    this.error = error;
  }
  update(f) {
    return this;
  }
}
```

### 4. Make error handling explicit in what objects are being constructed

Move the construction routine for Score and ScoreError into it's own function.

```

class Score {
  constructor(num, predicate, constraint) {
    this.num = num;
    this.predicate = predicate;
    this.constraint = constraint;
  }
  update(f) {
    let { num, predicate, constraint } = this;
    if (predicate(f(num))) {
      if (constraint(f(num))) {
        this.num = f(num)
      } else {
        return new ScoreError('Score not in valid range score range')
      }
    }
  }
}

class ScoreError {
  constructor(error) {
    this.error = error;
  }
  update(f) {
    return this;
  }
}

let isNumber = a => typeof a === 'number' && !isNaN(a);
let isInRange = n => n >= 0 && n <= 99;

let createScore = a => isNumber(a)
  ? isInRange(a)
    ? new Score(a, isNumber, isInRange)
    : new ScoreError('Value outside of score range')
  : new ScoreError('Value must be a number')

let inc = n => n+1;
let dec = n => n-2;
let double = n => n*2

let myScore = createScore(23, isNumber, isInRange)

myScore.update(double);

console.log(
  myScore
)
```
### 5. Reuse the construction routine for post update validation

Construction routine is already doing the validation for us, we can just reuse the routine now that it's a normal function outside the created objects

```

class Score {
  constructor(num) {
    this.num = num;
  }
  update(f) {
    return createScore(f(this.num))
  }
}

class ScoreError {
  constructor(error) {
    this.error = error;
  }
  update(f) {
    return this;
  }
}

let isNumber = a => typeof a === 'number' && !isNaN(a);
let isInRange = n => n >= 0 && n <= 99;

let createScore = a => isNumber(a)
  ? isInRange(a)
    ? new Score(a)
    : new ScoreError('Value outside of score range')
  : new ScoreError('Value must be a number')

let inc = n => n+1;
let dec = n => n-2;
let double = n => n*2

let myScore = createScore(23)

console.log(
  myScore.update(double)
)
```
### 6. We have a valid set of functors, renaming and adding methods

- .map to replace .update for common universal call syntax for all functors (like list functor for empty list and non-empty list)
- .fold as "get" method for grabbing the value and releasing the interface

```

class Score {
  constructor(num) {
    this.num = num;
  }
  map(f) {
    return createScore(f(this.num))
  }
  fold(f) {
    return f(this.num)
  }
}

class ScoreError {
  constructor(error) {
    this.error = error;
  }
  map(f) {
    return this;
  }
  fold(f) {
    return this.error
  }
}

let isNumber = a => typeof a === 'number' && !isNaN(a);
let isInRange = n => n >= 0 && n <= 99;

let createScore = a => isNumber(a)
  ? isInRange(a)
    ? new Score(a)
    : new ScoreError('Value outside of score range')
  : new ScoreError('Value must be a number')

let inc = n => n+1;
let dec = n => n-2;
let double = n => n*2

let myScore = createScore(23)

console.log(
  myScore.map(double).fold(a => a)
)
```

### 7. Add a way to concatenate scores with short circuit on Error

- Add .chain method that allows merging two score objects into a single score object

```

class Score {
  constructor(num) {
    this.num = num;
  }
  map(f) {
    return createScore(f(this.num))
  }
  fold(f) {
    return f(this.num)
  }
  chain(f) {
    return f(this.num)
  }
}

class ScoreError {
  constructor(error) {
    this.error = error;
  }
  map(f) {
    return this;
  }
  fold(f) {
    return this.error
  }
  chain(f) {
    return this;
  }
}

let isNumber = a => typeof a === 'number' && !isNaN(a);
let isInRange = n => n >= 0 && n <= 99;

let createScore = a => isNumber(a)
  ? isInRange(a)
    ? new Score(a)
    : new ScoreError('Value outside of score range')
  : new ScoreError('Value must be a number')

let inc = n => n+1;
let dec = n => n-2;
let double = n => n*2
let add = (a, b) => a+b;

let myScore = createScore(23)

let chain = f => (F, G) => F.chain(a => G.map(b => f(a, b)))

console.log(
  'case with errors: ', [5, null, 13]
    .map(createScore) // Turn list of inputs into list of Scores | ScoreErrors
    .map(ScoreSet => ScoreSet.map(inc)) // Increment Scores | ScoreErrors
    .reduce(chain(add)), // Concatenate into a single Score | ScoreError
)
console.log(
  'case without errors: ',  [3, 5, 7]
    .map(createScore) // Turn list of inputs into list of Scores | ScoreErrors
    .map(ScoreSet => ScoreSet.map(inc)) // Increment Scores | ScoreErrors
    .reduce(chain(add)) // Concatenate into a single Score | ScoreError
)
```

### Last notes since the .chain can be tough to digest

- Chain concatenates two objects belonging to the same the same set into an object of the same set.
- If the left hand functor F is an error object, the function from a => G.map is ignored and the return is just the functor F.
- If the left hand functor F hits a success case, the a => G.map gets discharged with a in scope, directly calling the right hand functor G's map method to get b into scope.
- If the right hand functor G hits an error case, the b => f(a, b) is ignored and the return is just the functor G.
- If the left hand and the right hand functors are both Score functors, then the both a => and b => functions get discharged and the right hand functor is reused with the construction routine post validation used in it's .map method
- For an algebraic set of functors the minimum viable product is
  - .of (construction routine)
  - .map (update)
  - .fold (get value and release the interface)
  -. chain (concatenate within the set)
