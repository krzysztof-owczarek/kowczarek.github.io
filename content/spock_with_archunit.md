## Using ArchUnit framework with Spock

Current versions of ArchUnit seem to be completely compatible with Spock Test Framework version 2.x.

Currently we do not need explicit JUnit dependency or JUnit runners.

Snippets:

```xml
<dependency>
    <groupId>org.spockframework</groupId>
    <artifactId>spock-core</artifactId>
    <version>2.0-M3-groovy-3.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.codehaus.groovy</groupId>
    <artifactId>groovy</artifactId>
    <version>3.0.5</version>
</dependency>
<dependency>
    <groupId>com.tngtech.archunit</groupId>
    <artifactId>archunit</artifactId>
    <version>0.14.1</version>
    <scope>test</scope>
</dependency>
```

```groovy
class ArchSpec extends Specification {

    def "no classes from package model should have deps on classes in package impl"() {
        given:
        def allClasses = new ClassFileImporter().importPackages("com.example.app")
        def rule = ArchRuleDefinition.noClasses()
                .that().resideInAPackage("..model..").should().dependOnClassesThat().resideInAPackage("..impl..")

        expect:
        rule.check(allClasses)
    }
}
```

...and that's it.