# Contracts
Contracts are used to allow the programmer to cooperate with Kotlin compiler by providing it with additional guarantees,
getting more complete and intense analysis in return.
`contract` is a function that takes a lambda expression as parameter and each line of the lambda passed to `contract`,
describes one effect of the function.

More detailed information about how contracts work can be found 
[here](https://github.com/Kotlin/KEEP/blob/master/proposals/kotlin-contracts.md).


## Encoding simple effects
Actually there are only 5 different simple effects which can be easily encoded as post conditions in the following way:
- `returns()` &rarr; Since we are not interested in proving termination, this effects does not give us any information
  and can be ignored
- `returnsNotNull()` &rarr; `ensures ret != null`
- `returns(true)` &rarr; `ensures ret == true`
- `returns(false)` &rarr; `ensures ret == false`
- `returns(null)` &rarr; `ensures ret == null`

## Encoding conditional effects
Conditional effects can be obtained by attaching a boolean expression to another SimpleEffect effect
with the function implies.
The syntax is the following: `simpleEffect implies booleanExpression`
and can be encoded as `ensures simpleEffect ==> booleanExpression`

example: `returns(false) implies (b)` &rarr; `ensures ret == false ==> b`


## Encoding calls in place
TODO

## FirContractDescription structure
- A `FirSimpleFunction` contains the infos about the contracts in the field `contractDescription`
of type `FirResolvedContractDescription`
- `contractDescription.effects` is a list of `FirEffectDeclaration` which represent the effects of the contract
- An object of type `FirEffectDeclaration` has a field called `effect` of type `ConeEffectDeclaration`,
  in order to distinguish between different types of effects we have to do pattern matching on that field:
  - is `ConeReturnsEffectDeclaration` &rarr; the effect is a `SimpleEffect` and we can understand which kind of
    `SimpleEffect` from the field `value`, in particular
    - `returns()` has `value = "ConeContractConstantValues.WILDCARD"`
    - `returnsNotNull()` has `value = "ConeContractConstantValues.NOT_NULL"`
    - `returns(true)` has `value = "ConeContractConstantValues.TRUE"`
    - `returns(false)` has `value = "ConeContractConstantValues.FALSE"`
    - `returns(null)` has `value = "ConeContractConstantValues.NULL"`
  - is `ConeConditionalEffectDeclaration` &rarr; the effect is a `ConditionalEffects`
    - `effect.effect.value.name` contains the value of the `SimpleEffect` which is the left part of the implication
    - `effect.condition` contains the boolean expression which is the right part of the implication
      (even though `implies` takes a `Boolean` argument, actually only a subset of valid Kotlin expressions is accepted:
      namely, null-checks (`== null`, `!= null`), instance-checks (`is`, `!is`), logic operators (`&&`, `||`, `!`)
  - is `ConeCallsEffectDeclaration` &rarr; the effect is `CallsInPlace`
