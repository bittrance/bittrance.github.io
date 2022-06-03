IMO calling this feature nice-to-have is underselling it. rspec, the original popularizer of BDD has this feature as a central concept, where it is called "subject" (though the docs make a poor job of explaining it). Fundamentally, subject (and ginko's justBeforeEach) allows defining the unit under test. It solves a basic problem with TDD, namely that it is hard to understand exactly what is tested by a suite or group of tests.
```
@Test
function testCreate() {
}

@Test
function testCreateWithFoo() {
}

@Test
function testRefuseToCreateWithBar() {
}
```

https://github.com/mochajs/mocha/issues/4113