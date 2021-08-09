---
title: Тестирование контроллеров .net core
description: интеграционное End to End тестирование с возможностью подмены зависимостей
---

Поставим задачу создания интеграционных тестов для апи, возможно с подменой части зависимостей на заглушки. Для начала создадим проект .net core и посмотрим на код инициализации

```c#
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(webBuilder =>
                                  {
                                      webBuilder.UseStartup<Startup>();
                                  });
}
```

Будем использовать `Autofac` и `Autofac.Extensions.DependencyInjection` При этом традиционно конфигурация конейнера autofac прописывается в startup, но из-за бага .net core это не позволит переопределить зависимости при тестировании. Поэтому будем конфигурировать контейнер в Program.cs

```c#
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
    .UseServiceProviderFactory(new AutofacServiceProviderFactory())
    .ConfigureContainer<ContainerBuilder>(builder =>
                                          {
                                              builder.RegisterType<WeatherService>()
                                                  .AsImplementedInterfaces().InstancePerLifetimeScope();
                                          })
    .ConfigureWebHostDefaults(webBuilder => { webBuilder.UseStartup<Startup>(); });
```

Далее создадим тестовый проект xUnit, подключим зависимости `Microsoft.AspNet.WebApi.Client` и `FluentAssertions` Для упрощения переопределения зависимостей создадим класс

```c#
class TestWebApplicationFactory: WebApplicationFactory<Startup>
{
    private Action<ContainerBuilder> _setupMockServicesAction;

    public TestWebApplicationFactory WithMockServices(Action<ContainerBuilder> setupMockServices)
    {
        _setupMockServicesAction = setupMockServices;
        return this;
    }

    protected override IHost CreateHost(IHostBuilder builder)
    {
        builder.ConfigureContainer<ContainerBuilder>(containerBuilder =>
                                                     {
                                                         _setupMockServicesAction?.Invoke(containerBuilder);
                                                     });

        return base.CreateHost(builder);
    }
}
```

 Тогда можно будет создать  тест метода контроллера с переопределением зависимости 

```c#
[Fact]
public async Task WeatherControllerTest()
{
    var mock = new Mock<IWeatherService>();
    mock.Setup(e => e.Get()).Returns(new List<WeatherForecast>());

    using var testWebApplicationFactory = new TestWebApplicationFactory()
        .WithMockServices(containerBuilder =>
                          {
                              containerBuilder.Register(s => mock.Object).AsImplementedInterfaces()
                                  .InstancePerLifetimeScope();
                          });

    var resp = await testWebApplicationFactory.CreateClient().GetAsync("/weatherforecast");

    resp.StatusCode.Should().Be(HttpStatusCode.OK);
    var result = await resp.Content.ReadAsStringAsync();
    result.Should().Be("[]");

    mock.Verify(m=>m.Get(), Times.AtLeastOnce);
}
```



> Использованные материалы
>
> https://github.com/autofac/Autofac/issues/1207
>
> https://github.com/dotnet/aspnetcore/blob/a4d9faefe2ed6fbebd500413032336607ec92778/src/Hosting/TestHost/test/TestServerTests.cs#L51-L64
>
> https://andrewlock.net/converting-integration-tests-to-net-core-3/