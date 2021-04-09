### Maven repository injection vulnerability

This project demonstrates that all Apache Maven™ projects are vulnerable to repository injection.

The POC consists of several modules organized in a single Maven build for practical reasons but would be separate in a read-world attack:

- the main module (root) just aggregates the different parts of the POC
- the `app` module demonstrates a vulnerable build
  - this build has a dependency on the `innocent` module
  - as well as a dependency on Apache Commons Lang3
  - it uses code from Apache Lang3 and expects that the dependency will be fetched from Maven Central
- the `innocent`  module is the attacker dependency
  - it declares a dependency on Apache Commons Lang3
  - and a _custom repository_ which will contain a malicious version of Apache Commons Lang3
- the `malicious` module impersonates Apache Lang3
  - it implements its API, making it impossible for the victim to figure out that it was substituted
  - introduces malicious code
  - is published in the malicious repository
  
### Proof-of-concept

Note that the commands below use `-gs ../settings.xml` to use an alternate settings file, in order to make sure that we don't use the local `.m2` cache for the demonstration.

The first thing to do is to populate the malicious repository with the malicious dependency:

    cd malicious
    mvn -gs ../settings.xml deploy

Then publish the attacker library:

    cd innocent
    mvn -gs ../settings2.xml install    
    
Then go to the `app` victim directory and execute the application:

    cd ../app
    mvn -gs ../settings2.xml compile
    mvn -gs ../settings2.xml exec:java
    
The output will be:

    You've been pwned! Hello, world!
    
Instead of the expected:

    Hello, World!
    
### Why?

Apache Maven blindly uses repositories defined by transitive dependencies.
Because Maven uses the nearest first strategy, the first dependency it sees in this build is `innocent`, which defines a malicious repository.
Then it sees the dependency on `org.apache.commons:commons-lang3` and tries to download it.
Because the dependency was made available via the `innocent` dependency and its vulnerable repository, the real Apache Commons Lang3 dependency is NOT fetched from Maven Central.

### Mitigation

This is a design flaw in Apache Maven: transitive repositories should never be used automatically.
Repositories should be redeclared on the _consumer_ side.
It is however possible to mitigate by defining a mirror for the malicious repository by pointing the mirror to a sane repository.

It's worth noting that if the dependency was previously fetched into the local `.m2` repository, then Maven would not try to download it from the malicious repository.
