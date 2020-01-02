# Background tasks with hosted services

This is fully described in [this msdn article](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services).

## So what's the problem?

Sometimes I just want to create a class that basically only gets instantiated to hook into some event and waits for it to trigger.

For example: In a discord bot, this could be a special pipeline that waits for a message received event.

Normally, I'd just inject this class into my "root" class as a dependency and leave it there.

```cs
public sealed class DraxisBot
{
    public DraxisBot(WaitingRoomService waitingRoomService)
    {
        // do nothing with waitingRoomService, since it hooks into
        // message received when constructed.

        // Visual Studio complains that I'm not using waitingRoomService.
    }
}
```

## How do we solve this?

Apparently, there is a concept of a "hosted service". What this basically means is that we can tell our inversion of control container (be it ASP.NET DI or in my case Ninject) that a certain class is registered as a hosted service.

The inversion of control container then makes sure to instantiate it when your app boots up, and clear it after it's done.

Pretty neat.

## What about implementation?

In ASP.NET, you'd register your class as a hosted service:

```cs
services.AddHostedService<WelcomeMessageService>();
```

It would also implement IHostedService. Ninject, however, doesn't have the ability to add hosted services (there probably is an extension somewhere).

Since we cannot register it as the hosted service, we can attach an OnActivation method on top of our root object that requests those two services.

```cs
Bind<DraxisBot>().ToSelf().OnActivation(bot =>
    {
        Kernel.Get<WelcomeMessageService>();
        Kernel.Get<UsernameService>();
    });
```

This approach gets rid of the "waitingRoomService is not used" problem and moves the dependencies configuration into the NinjectModule.

When the activation method runs, it requests our hosted services. It does, however, garbage collect them after this method is done, since we don't store the references anywhere.

**For that reason, we need to register these services as singletons.**

This is much better than just requesting dependencies to not use them. :)
