
### Introduction
The concept of Dependency Injection is, at its core, a fundamentally simple notion. It is, however, commonly presented in a manner alongside the more theoretical concepts of Inversion of Control, Dependency Inversion, the SOLID Principles, and so forth. To make it as easy as possible for you to get started using Dependency Injection and begin reaping its benefits, this article will remain very much on the practical side of the story, depicting examples that show precisely the benefits of its use, in a manner chiefly divorced from the associated theory. We'll spend only a very little amount of time discussing the academic concepts that surround dependency injection here, for the bulk of that explanation will be reserved for the second article of this series. Indeed, entire books can be and have been written that provide a more in-depth and rigorous treatment of the concepts. 

Here, we'll start with a simple explanation, move to a few more real-world examples, and then discuss some background information. Another article, to follow this one, will discuss how Dependency Injection fits into the overall ecosystem of applying best-practice architectural patterns.

### A Simple Explanation
"Dependency Injection" is an overly-complex term for an extremely simple concept. At this point, some wise and reasonable questions would be "how do you define "dependency"?", "what does it mean for a dependency to be "injected"?", "can you inject dependencies in different ways?", "why is this useful?" You might not believe that a term like "Dependency Injection" can be explained in two code snippets and a couple of words, but alas, it can:

The simplest way to explain the concept is to show you. This is **not** dependency injection:

```typescript
import { Engine } from './Engine';

class Car {
    private engine: Engine;

    public constructor () {
        this.engine = new Engine();
    }

    public startEngine(): void {
        this.engine.fireCylinders();
    }
}
```
And this **is** dependency injection:

```typescript
import { Engine } from './Engine';

class Car {
    private engine: Engine;

    public constructor (engine: Engine) {
        this.engine = engine;
    }
    
    public startEngine(): void {
        this.engine.fireCylinders();
    }
}
```
Done. That's it. Cool. The end.

What changed? Rather than allow the `Car` class to instantiate `Engine`, as it did in the first example, in the second example, `Car` had an instance of `Engine` passed in, or *injected* in, from some higher level of control, to its constructor. That's it. At its core, this is all dependency injection is - the act of injecting (passing) a dependency into another class or function. Anything else involving the notion of dependency injection is simply a variation on this fundamental and simple concept. Put trivially, dependency injection is a technique whereby an object receives other objects it depends on, called dependencies, rather than creating them itself.

In general, to define what a "dependency" is, if some class `A` uses the functionality of a class `B`, then `B` is a dependency for `A`, or, in other words, `A` has a dependency on `B`. Of course, this isn't limited to classes and holds for functions too. In this case, the class `Car` has a dependency on the `Engine` class, or `Engine` is a dependency of `Car`. Dependencies are simply variables, just like most things in programming.

Dependency Injection is widely-used to support many use cases, but perhaps the most blatant of uses is to permit easier testing. In the first example, we can't easily mock out `engine` because the `Car` class instantiates it. The real engine is always being used. But, in the latter case, we have control over the `Engine` that is used, which means, in a test, we can subclass `Engine` and override its methods. For example, if we wanted to see what `Car.startEngine()` does if `engine.fireCylinders()` throws an error, we could simply create a `FakeEngine` class, have it extend the `Engine` class, and then override `fireCylinders` to make it throw an error. In the test, we can inject that `FakeEngine` object into the constructor for `Car`. Since `FakeEngine` **is an** `Engine` by implication of inheritance, the TypeScript type system is satisfied.

I want to make it very, very clear that what you see above is the core notion of dependency injection. A `Car`, by itself, is not smart enough to know what engine it needs. Only the engineers that *construct* the car understand the requirements for its engines and wheels. Thus, it makes sense that the people who *construct* the car provide the specific engine required, rather than letting a `Car` itself pick whichever engine it wants to use. I use the word "construct" specifically because you construct the car by calling the constructor, which is the place dependencies are injected. If the car also created its own tires in addition to the engine, how do we know that the tires being used are safe to be spun at the max RPM the engine can output? For all these reasons and more, it should make sense, perhaps intuitively, that `Car` should have nothing to do with deciding what `Engine` and what `Wheels` it uses. They should be provided from some higher level of control. In the latter example depicting dependency injection in action, if you imagine `Engine` to be an abstract class rather than a concrete one, this should make even more sense - the car knows it needs an engine and it knows the engine has to have some basic functionality, but how that engine is managed and what the specific implementation of it is is reserved for being decided and provided by the piece of code that creates (constructs) the car.

### A Real-World Example
We're going to look at a few more practical examples that hopefully help to explain, again intuitively, why dependency injection is useful. Hopefully, by not harping on the theoretical and instead moving straight into applicable concepts, you can more fully see the benefits that dependency injection provides, and the difficulties of life without it. We'll revert to a slightly more "academic" treatment of the topic later.

We'll start by constructing our application normally, in a manner highly coupled, without utilizing dependency injection or abstractions, so that we come to see the downsides of this approach and the difficulty it adds to testing. Along the way, we'll gradually refactor until we rectify all of the issues.

To begin, suppose you've been tasked with building two classes - an email provider and a class for a data access layer that needs to be used by some `UserService`. We'll start with data access, but both are easily defined:

```typescript
// UserRepository.ts

import { dbDriver } from 'pg-driver';

export class UserRepository {
    public async addUser(user: User): Promise<void> {
        // ... dbDriver.save(...)
    }

    public async findUserById(id: string): Promise<User> {
        // ... dbDriver.query(...)
    }
    
    public async existsByEmail(email: string): Promise<boolean> {
        // ... dbDriver.save(...)
    }
}
```
***Note:** The name "Repository" here comes from the "Repository Pattern", a method of decoupling your database from your business logic. You can [learn more about the Repository Pattern](https://martinfowler.com/eaaCatalog/repository.html), but for the purposes of this article, you can simply consider it to be some class that encapsulates away your database so that, to business logic, your data storage system is treated as merely an in-memory collection. Explaining the Repository Pattern fully is outside the purview of this article.*

This is how we normally expect things to work, and `dbDriver` is hardcoded within the file. 

In your `UserService`, you'd import the class, instantiate it, and start using it:

```typescript
import { UserRepository } from './UserRepository.ts';

class UserService {
    private readonly userRepository: UserRepository;
    
    public constructor () {
        // Not dependency injection.
        this.userRepository = new UserRepository();
    }

    public async registerUser(dto: IRegisterUserDto): Promise<void> {
        // User object & validation
        const user = User.fromDto(dto);

        if (await this.userRepository.existsByEmail(dto.email))
            return Promise.reject(new DuplicateEmailError());
            
        // Database persistence
        await this.userRepository.addUser(user);
        
        // Send a welcome email
        // ...
    }

    public async findUserById(id: string): Promise<User> {
        // No need for await here, the promise will be unwrapped by the caller.
        return this.userRepository.findUserById(id);
    }
}
```
Once again, all remains normal.

***A brief aside:** A DTO is a Data Transfer Object - it's an object that acts as a property bag to define a standardized data shape as it moves between two external systems or two layers of an application. You can learn more about DTOs from Martin Fowler's article on the topic, [here](https://martinfowler.com/eaaCatalog/dataTransferObject.html). In this case, `IRegisterUserDto` defines a contract for what the shape of data should be as it comes up from the client. I only have it contain two properties - `id` and `email`. You might think it's peculiar that the DTO we expect from the client to create a new user contains the user's ID even though we haven't created a user yet. The ID is a UUID and I allow the client to generate it for a variety of reasons, which are outside the scope of this article. Additionally, the `findUserById` function should map the `User` object to a response DTO, but I neglected that for brevity. Finally, in the real world, I wouldn't have a `User` domain model contain a `fromDto` method. That's not good for domain purity. Once again, its purpose is brevity here.*

Next, you want to handle the sending of emails. Once again, as normal, you can simply create an email provider class and import it into your `UserService`.

```typescript
// SendGridEmailProvider.ts

import { sendMail } from 'sendgrid';

export class SendGridEmailProvider {
    public async sendWelcomeEmail(to: string): Promise<void> {
        // ... await sendMail(...);
    }
}
```
Within `UserService`:

```typescript
import { UserRepository }  from  './UserRepository.ts';
import { SendGridEmailProvider } from './SendGridEmailProvider.ts';

class UserService {
    private readonly userRepository: UserRepository;
    private readonly sendGridEmailProvider: SendGridEmailProvider;

    public constructor () {
        // Still not doing dependency injection.
        this.userRepository = new UserRepository();
        this.sendGridEmailProvider = new SendGridEmailProvider();
    }

    public async registerUser(dto: IRegisterUserDto): Promise<void> {
        // User object & validation
        const user = User.fromDto(dto);
        
        if (await this.userRepository.existsByEmail(dto.email))
            return Promise.reject(new DuplicateEmailError());
        
        // Database persistence
        await this.userRepository.addUser(user);
        
        // Send welcome email
        await this.sendGridEmailProvider.sendWelcomeEmail(user.email);
    }

    public async findUserById(id: string): Promise<User> {
        return this.userRepository.findUserById(id);
    }
}
```
We now have a fully working class, and in a world where we don't care about testability or writing clean code by any manner of the definition at all, and in a world where technical debt is non-existent and pesky program managers don't set deadlines, this is perfectly fine. Unfortunately, that's not a world we have the benefit of living in. 

What happens when we decide we need to migrate away from SendGrid for emails and use MailChimp instead? Similarly, what happens when we want to unit test our methods - are we going to use the real database in the tests? Worse, are we actually going to send real emails to potentially real email addresses and pay for it too? In the traditional JavaScript ecosystem, the methods of unit testing classes under this configuration are fraught with complexity and over-engineering. People bring in whole entire libraries simply to provide stubbing functionality, which adds all kinds of layers of indirection, and, even worse, can directly couple the tests to the implementation of the system under test, when, in reality, tests should never know how the real system works (this is known as black-box testing). We'll work to mitigate these issues as we discuss what the actual responsibility of `UserService` is and apply new techniques of dependency injection.

Consider, for a moment, what a `UserService` does. The whole point of the existence of `UserService` is to execute specific use cases involving users - registering them, reading them, updating them, etc. It's a best practice for classes and functions to have only one responsibility (SRP - the Single Responsibility Principle), and the responsibility of `UserService` is to handle user-related operations. Why, then, is `UserService` responsible for controlling the lifetime of `UserRepository` and `SendGridEmailProvider` in this example? Imagine if we had some other class used by `UserService` which opened a long-running connection. Should `UserService` be responsible for disposing of that connection too? Of course not. All of these dependencies have a lifetime associated with them - they could be singletons, they could be transient and scoped to a specific HTTP Request, etc. The controlling of these lifetimes is well outside the purview of `UserService`. So, to solve these issues, we'll inject all of the dependencies in, just like we saw before.

```typescript
import { UserRepository }  from  './UserRepository.ts';
import { SendGridEmailProvider } from './SendGridEmailProvider.ts';

class UserService {
    private readonly userRepository: UserRepository;
    private readonly sendGridEmailProvider: SendGridEmailProvider;

    public constructor (
        userRepository: UserRepository,
        sendGridEmailProvider: SendGridEmailProvider
    ) {
        // Yay! Dependencies are injected.
        this.userRepository = userRepository;
        this.sendGridEmailProvider = sendGridEmailProvider;
    }

    public async registerUser(dto: IRegisterUserDto): Promise<void> {
        // User object & validation
        const user = User.fromDto(dto);

        if (await this.userRepository.existsByEmail(dto.email))
            return Promise.reject(new DuplicateEmailError());
        
        // Database persistence
        await this.userRepository.addUser(user);
        
        // Send welcome email
        await this.sendGridEmailProvider.sendWelcomeEmail(user.email);
    }

    public async findUserById(id: string): Promise<User> {
        return this.userRepository.findUserById(id);
    }
}
```
Great - now `UserService` receives pre-instantiated objects, and whichever piece of code calls and creates a new `UserService` is the piece of code in charge of controlling the lifetime of the dependencies. We've inverted control away from `UserService` and up to a higher level. If I only wanted to show how we could inject dependencies through the constructor as to explain the basic tenant of dependency injection, I could stop here. There are still some problems from a design perspective, however, which when rectified, will serve to make our use of dependency injection all the more powerful.

Firstly, why does `UserService` know that we're using SendGrid for emails? Secondly, both dependencies are on concrete classes - the concrete `UserRepository` and the concrete `SendGridEmailProvider`. This relationship is too rigid - we're stuck having to pass in some object that **is a** `UserRepository` and **is a** `SendGridEmailProvider`.

This isn't great because we want `UserService` to be completely agnostic to the implementation of its dependencies. By having `UserService` be blind in that manner, we can swap out the implementations without affecting the service at all - this means, if we decide to migrate away from SendGrid and use MailChimp instead, we can do so. It also means if we want to fake out the email provider for tests, we can do that too. What would be useful is if we could define some public interface and force that incoming dependencies abide by that interface, while still having `UserService` be agnostic to implementation details. Put another way, we need to force `UserService` to only depend on an abstraction of its dependencies, and not it's actual concrete dependencies. We can do that through, well, interfaces. 

Start by defining an interface for the `UserRepository` and implement it:

```typescript
// UserRepository.ts

import { dbDriver } from 'pg-driver';

export interface IUserRepository {
    addUser(user: User): Promise<void>;
    findUserById(id: string): Promise<User>;
    existsByEmail(email: string): Promise<boolean>;
}

export class UserRepository implements IUserRepository {
    public async addUser(user: User): Promise<void> {
        // ... dbDriver.save(...)
    }

    public async findUserById(id: string): Promise<User> {
        // ... dbDriver.query(...)
    }

    public async existsByEmail(email: string): Promise<boolean> {
        // ... dbDriver.save(...)
    }
}
```
And define one for the email provider, also implementing it:

```typescript
// IEmailProvider.ts
export interface IEmailProvider {
    sendWelcomeEmail(to: string): Promise<void>;
}

// SendGridEmailProvider.ts
import { sendMail } from 'sendgrid';
import { IEmailProvider } from './IEmailProvider';

export class SendGridEmailProvider implements IEmailProvider {
    public async sendWelcomeEmail(to: string): Promise<void> {
        // ... await sendMail(...);
    }
}
```
***Note:** This is the [Adapter Pattern](https://en.wikipedia.org/wiki/Adapter_pattern) from the Gang of Four Design Patterns.*

Now, our `UserService` can depend on the interfaces rather than the concrete implementations of the dependencies:

```typescript
import { IUserRepository }  from  './UserRepository.ts';
import { IEmailProvider } from './SendGridEmailProvider.ts';

class UserService {
    private readonly userRepository: IUserRepository;
    private readonly emailProvider: IEmailProvider;

    public constructor (
        userRepository: IUserRepository,
        emailProvider: IEmailProvider
    ) {
        // Double yay! Injecting dependencies and coding against interfaces.
        this.userRepository = userRepository;
        this.emailProvider = emailProvider;
    }

    public async registerUser(dto: IRegisterUserDto): Promise<void> {
        // User object & validation
        const user = User.fromDto(dto);

        if (await this.userRepository.existsByEmail(dto.email))
            return Promise.reject(new DuplicateEmailError());
        
        // Database persistence
        await this.userRepository.addUser(user);
        
        // Send welcome email
        await this.emailProvider.sendWelcomeEmail(user.email);
    }

    public async findUserById(id: string): Promise<User> {
        return this.userRepository.findUserById(id);
    }
}
```
If interfaces are new to you, this might look very, very complex. Indeed, the concept of building loosely coupled software might be new to you too. Think about wall receptacles. You can plug any device into any receptacle so long as the plug fits the outlet. That's loose coupling in action. Your toaster is not hard-wired into the wall, because if it was, and you decide to upgrade your toaster, you're out of luck. Instead, outlets are used, and the outlet defines the interface. Similarly, when you plug an electronic device into your wall receptacle, you're not concerned with the voltage level, the max current draw, the AC frequency, etc., you just care if the plug fits into the outlet. You could have an electrician come in and change all the wires behind that outlet, and you won't have any problems plugging in your toaster, so long as that outlet doesn't change. Further, your electricity source could be switched to come from the city or your own solar panels, and once again, you don't care as long as you can still plug into that outlet. The interface is the outlet, providing "plug-and-play" functionality. In this example, the wiring in the wall and the electricity source is akin to the dependencies and your toaster is akin to the `UserService` (it has a dependency on the electricity) - the electricity source can change and the toaster still works fine and need not be touched, because the outlet, acting as the interface, defines the standard means for both to communicate. In fact, you could say that the outlet acts as an "abstraction" of the wall wiring, the circuit breakers, the electrical source, etc.

It is a common and well-regarded principle of software design, for the reasons above, to code against interfaces (abstractions) and not implementations, which is what we've done here. In doing so, we're given the freedom to swap out implementations as we please, for those implementations are hidden behind the interface (just like wall wiring is hidden behind the outlet), and so the business logic that uses the dependency never has to change so long as the interface never changes. Remember, `UserService` only needs to know what functionality is offered by its dependencies, not how that functionality is supported behind the scenes. That's why using interfaces works. 

These two simple changes of utilizing interfaces and injecting dependencies make all the difference in the world when it comes to building loosely coupled software and solves all of the problems we ran into above.

If we decide tomorrow that we want to rely on MailChimp for emails, we simply create a new MailChimp class that honors the `IEmailProvider` interface and inject it in instead of SendGrid. The actual `UserService` class never has to change even though we've just made a ginormous change to our system by switching to a new email provider. The beauty of these patterns is that `UserService` remains blissfully unaware of how the dependencies it uses work behind the scenes. The interface serves as the architectural boundary between both components, keeping them appropriately decoupled.

Additionally, when it comes to testing, we can create fakes that abide by the interfaces and inject them instead. Here, you can see a fake repository and a fake email provider.

```typescript
// Both fakes:
class FakeUserRepository implements IUserRepository {
    private readonly users: User[] = [];

    public async addUser(user: User): Promise<void> {
        this.users.push(user);
    }

    public async findUserById(id: string): Promise<User> {
        const userOrNone = this.users.find(u => u.id === id);

        return userOrNone
            ? Promise.resolve(userOrNone)
            : Promise.reject(new NotFoundError());
    }

    public async existsByEmail(email: string): Promise<boolean> {
        return Boolean(this.users.find(u => u.email === email));
    }

    public getPersistedUserCount = () => this.users.length;
}

class FakeEmailProvider implements IEmailProvider {
    private readonly emailRecipients: string[] = [];

    public async sendWelcomeEmail(to: string): Promise<void> {
        this.emailRecipients.push(to);
    }

    public wasEmailSentToRecipient = (recipient: string) =>
        new Boolean(this.emailRecipients.find(r => r === recipient));
}
```
Notice that both fakes implement the same interfaces that `UserService` expects its dependencies to honor. Now, we can pass these fakes into `UserService` instead of the real classes and `UserService` will be none the wiser, it'll use them just as if they were the real deal. The reason it can do that is because it knows that all of the methods and properties it wants to use on its dependencies do indeed exist and are indeed accessible (because they implement the interfaces), which is all `UserService` needs to know (i.e, not how the dependencies work).

We'll inject these two during tests, and it'll make the testing process so much easier and so much more straight forward than what you might be used to when dealing with over-the-top mocking and stubbing libraries, working with Jest's own internal tooling, or trying to monkey-patch.

Here are actual tests using the fakes.
```typescript
// Fakes
let fakeUserRepository: FakeUserRepository;
let fakeEmailProvider: FakeEmailProvider;

// SUT
let userService: UserService;

// We want to clean out the internal arrays of both fakes 
// before each test.
beforeEach(() => {
    fakeUserRepository = new FakeUserRepository();
    fakeEmailProvider = new FakeEmailProvider();
    
    userService = new UserService(fakeUserRepository, fakeEmailProvider);
});

// A factory to easily create DTOs.
// Here, we have the optional choice of overriding the defaults
// thanks to the built in `Partial` utility type of TypeScript.
function createSeedRegisterUserDto(opts?: Partial<IRegisterUserDto>): IRegisterUserDto {
    return {
        id: 'someId',
        email: 'example@domain.com',
        ...opts
    };
}

test('should correctly persist a user and send an email', async () => {
    // Arrange
    const dto = createSeedRegisterUserDto();

    // Act
    await userService.registerUser(dto);

    // Assert
    const expectedUser = User.fromDto(dto);
    const persistedUser = await fakeUserRepository.findUserById(dto.id);
    
    const wasEmailSent = fakeEmailProvider.wasEmailSentToRecipient(dto.email);

    expect(persistedUser).toEqual(expectedUser);
    expect(wasEmailSent).toBe(true);
});

test('should reject with a DuplicateEmailError if an email already exists', async () => {
    // Arrange
    const existingEmail = 'john.doe@live.com';
    const dto = createSeedRegisterUserDto({ email: existingEmail });
    const existingUser = User.fromDto(dto);
    
    await fakeUserRepository.addUser(existingUser);

    // Act, Assert
    await expect(() => userService.registerUser(dto))
        .rejects.toBeInstanceOf(new DuplicateEmailError());

    expect(fakeUserRepository.getPersistedUserCount()).toBe(1);
});

test('should correctly return a user', async () => {
    // Arrange
    const user = User.fromDto(createSeedRegisterUserDto());
    await fakeUserRepository.addUser(user);

    // Act
    const receivedUser = await userService.findUserById(user.id);

    // Assert
    expect(receivedUser).toEqual(user);
});
```
You'll notice a few things here - the hand-written fakes are very simple. There's no complexity from mocking frameworks which only serve to obfuscate. Everything is hand-rolled and that means there is no magic in the codebase. Asynchronous behavior is faked to match the interfaces. I use async/await in the tests even though all behavior is synchronous because I feel that it more closely matches how I'd expect the operations to work in the real world and because by adding async/await, I can run this same test suite against real implementations too in addition to the fakes, thus handing asynchrony appropriately is required. In fact, in real life, I would most likely not even worry about mocking the database and would instead use a local DB in a Docker container until there were so many tests that I had to mock it away for performance. I could then run the in-memory DB tests after every single change and reserve the real local DB tests for right before committing changes and for on the build server in the CI/CD pipeline.

In the first test, in the "arrange" section, we simply create the DTO. In the "act" section, we call the system under test and execute its behavior. Things get slightly more complex when making assertions. Remember, at this point in the test, we don't even know if the user was saved correctly. So, we define what we expect a persisted user to look like, and then we call the fake Repository and ask it for a user with the ID we expect. If the `UserService` didn't persist the user correctly, this will throw a `NotFoundError` and the test will fail, otherwise, it will give us back the user. Next, we call the fake email provider and ask it if it recorded sending an email to that user. Finally, we make the assertions with Jest and that concludes the test. It's expressive and reads just like how the system is actually working. There's no indirection from mocking libraries and there's no coupling to the implementation of the `UserService`.

In the second test, we create an existing user and add it to the repository, then we try to call the service again using a DTO that has already been used to create and persist a user, and we expect that to fail. We also assert that no new data was added to the repository. 

For the third test, the "arrange" section now consists of creating a user and persisting it to the fake Repository. Then, we call the SUT, and finally, check if the user that comes back is the one we saved in the repo earlier. 

These examples are relatively simple, but when things get more complex, being able to rely on dependency injection and interfaces in this manner keeps your code clean and make writing tests a joy. 

***A brief aside on testing:** In general, you don't need to mock out every dependency that the code uses. Many people, erroneously, claim that a "unit" in a "unit test" is one function or one class. That could not be more incorrect. The "unit" is defined as the "unit of functionality" or the "unit of behavior", not one function or class. So if a unit of behavior uses 5 different classes, you don't need to mock out all those classes *unless* they reach outside of the boundary of the module. In this case, I mocked the database and I mocked the email provider because I have no choice. If I don't want to use a real database and I don't want to send an email, I have to mock them out. But if I had a bunch more classes that didn't do anything across the network, I would not mock them because they're implementation details of the unit of behavior. I could also decide against mocking the database and emails and spin up a real local database and a real SMTP server, both in Docker containers. On the first point, I have no problem using a real database and still calling it a unit test so long as it's not too slow. Generally, I'd use the real DB first until it became too slow and I had to mock, as discussed above. But, no matter what you do, you have to pragmatic - sending welcome emails is not a mission-critical operation, thus we don't need to go that far in terms of SMTP servers in Docker containers. Whenever I do mock, I would be very unlikely to use a mocking framework or try to assert on the number of times called or parameters passed except in very rare cases, because that would couple tests to the implementation of the system under test, and they should be agnostic to those details.*

### Performing Dependency Injection without Classes and Constructors

So far, throughout the article, we've worked exclusively with classes and injected the dependencies through the constructor. If you're taking a functional approach to development and wish not to use classes, one can still obtain the benefits of dependency injection using function arguments. For example, our `UserService` class above could be refactored into:

```typescript
async function makeUserService(
    userRepository: IUserRepository,
    emailProvider: IEmailProvider
): IUserService {
    return {
        registerUser: dto => {
            // ...
        },

        findUserById: id => userRepository.findById(id)
    }
}
```
It's a factory that receives the dependencies and constructs the service object. We can also inject dependencies into Higher Order Functions. A typical example would be creating an Express Middleware function that gets a `UserRepository` and an `ILogger` injected:

```typescript
function authProvider(userRepository: IUserRepository, logger: ILogger) {
    return async (req: Request, res: Response, next: NextFunction) => {
        // ...
        // Has access to userRepository, logger, req, res, and next.
    }
}
```
In the first example, I didn't define the type of `dto` and `id` because if we define an interface called `IUserService` containing the method signatures for the service, then the TS Compiler will infer the types automatically. Similarly, had I defined a function signature for the Express Middleware to be the return type of `authProvider`, I wouldn't have had to declare the argument types there either.

If we considered the email provider and the repository to be functional too, and if we injected their specific dependencies as well instead of hard coding them, the root of the application could look like this:

```typescript
import { sendMail } from 'sendgrid';

async function main() {
    const app = express();
    
    const dbConnection = await connectToDatabase();
    
    // Change emailProvider to `makeMailChimpEmailProvider` whenever we want
    // with no changes made to dependent code.
    const userRepository = makeUserRepository(dbConnection);
    const emailProvider = makeSendGridEmailProvider(sendMail);
    
    const userService = makeUserService(userRepository, emailProvider);

    // Put this into another file. It's a controller action.
    app.post('/login', (req, res) => {
        await userService.registerUser(req.body as IRegisterUserDto);
        return res.send();
    });

    // Put this into another file. It's a controller action.
    app.delete(
        '/me', 
        authProvider(userRepository, emailProvider), 
        (req, res) => { ... }
    );
}
```
Notice that we fetch the dependencies that we need, like a database connection or third-party library functions, and then we utilize factories to make our first-party dependencies using the third-party ones. We then pass them into the dependent code. Since everything is coded against abstractions, I can swap out either `userRepository` or `emailProvider` to be any different function or class with any implementation I want (that still implements the interface correctly) and `UserService` will just use it with no changes needed, which, once again, is because `UserService` cares about nothing but the public interface of the dependencies, not how the dependencies work.

As a disclaimer, I want to point out a few things. As stated earlier, this demo was optimized for showing how dependency injection makes life easier, and thus it wasn't optimized in terms of system design best practices insofar as the patterns surrounding how Repositories and DTOs should technically be used. In real life, one has to deal with managing transactions across repositories and the DTO should generally not be passed into service methods, but rather mapped in the controller to allow the presentation layer to evolve separately from the application layer. The `userSerivce.findById` method here also neglects to map the User domain object to a DTO, which it should do in real life. None of this affects the DI implementation though, I simply wanted to keep the focus on the benefits of DI itself, not Repository design, Unit of Work management, or DTOs. Finally, although this may look a little like the NestJS framework in terms of the manner of doing things, it's not, and I actively discourage people from using NestJS for reasons outside the scope of this article.

###  A Brief Theoertical Overrview 
All applications are made up of collaborating components, and the manner in which those collaborators collaborate and are managed will decide how much the application will resist refactoring, resist change, and resist testing. Dependency injection mixed with coding against interfaces is a primary method (among others) of reducing the coupling of collaborators within systems, and making them easily swappable. This is the hallmark of a highly cohesive and loosely coupled design.

The individual components that make up applications in non-trivial systems must be decoupled if we want the system to be maintainable, and the way we achieve that level of decoupling, as stated above, is by depending upon abstractions, in this case, interfaces, rather than concrete implementations, and utilizing dependency injection. Doing so provides loose coupling and gives us the freedom of swapping out implementations without needing to make any changes on the side of the dependent component/collaborator and solves the problem that dependent code has no business managing the lifetime of its dependencies and shouldn't know how to create them or dispose of them.  

Despite the simplicity of what we've seen thus far, there's a lot more complexity that surrounds dependency injection.

Injection of dependencies can come in many forms. Constructor Injection is what we have been using here since dependencies are injected into a constructor. There also exists Setter Injection and Interface Injection. In the case of the former, the dependent component will expose a setter method which will be used to inject the dependency - that is, it could expose a method like `setUserRepository(userRepository: UserRepository)`. In the last case, we can define interfaces through which to perform the injection, but I'll omit the explanation of the last technique here for brevity since we'll spend more time discussing it and more in the second article of this series.

Because wiring up dependencies manually can be difficult, various IoC Frameworks and Containers exist. These containers store your dependencies and resolve the correct ones at runtime, often through Reflection in languages like C# or Java, exposing various configuration options for dependency lifetime. Despite the benefits that IoC Containers provide, there are cases to be made for moving away from them, and only resolving dependencies manually. To hear more about this, see Greg Young's [8 Lines of Code talk](https://www.infoq.com/presentations/8-lines-code-refactoring/).

Additionally, DI Frameworks and IoC Containers can provide too many options, and many rely on decorators or attributes to perform techniques such as setter or field injection. I look down on this kind of approach because, if you think about it intuitively, the point of dependency injection is to achieve loose coupling, but if you begin to sprinkle IoC Container-specific decorators all over your business logic, while you may have achieved decoupling from the dependency, you've inadvertently coupled yourself to the IoC Container. IoC Containers like Awilix solve this problem since they remain divorced from your application's business logic.

### Conclusion

This article served to depict only a very practical example of dependency injection in use and mostly neglected the theoretical attributes. I did it this way in order to make it easier to understand what dependency injection is at its core in a manner divorced from the rest of the complexity that people usually associate with the concept. In the second article of this series, we'll take a much, much more in-depth look, including at:

- The difference between Dependency Injection and Dependency Inversion and Inversion of Control
- Dependency Injection anti-patterns
- IoC Container anti-patterns
- The role of IoC Containers
- The different types of dependency lifetimes
- How IoC Containers are designed 
- Dependency Injection with React
- Advanced testing scenarios
- And more
