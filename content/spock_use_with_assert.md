## Spock Test Framework Tricks

### Use *with* to check object attributes for equality

```
given:
def employee = Employee.of("name", "surname", "phoneNumber")

expect:
with(employee) {
    name == "name"
    surname == "surname"
    phoneNumber == "phoneNumber"
}
```

This snippet should work as long as we use *with* in our expect or then block.

### What if we would like to extract the with-assertion logic to the external function?

```
def void test(Epmloyee employee) {
    with(employee) {
        name == "name"
        surname == "surname"
        phoneNumber == "phoneNumber"
    }
}

def "should check fields for equality"() {
    given:
    def employee = Employee.of("name", "surname", "phoneNumber")

    expect:
    test(employee)
}
```

The *void* keyword in test function declaration is important. Without it the function will try to return value from *with()* block and the result will be null.