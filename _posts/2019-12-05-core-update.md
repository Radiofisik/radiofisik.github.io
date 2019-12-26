---
title: Обновление на .Net Core 3
description: Возникшие проблемы и их решения
---

## Использование внутреннего API

Для получения логов запросов к API использовалось Middleware для логирования  https://radiofisik.ru/2019/07/04/logging/  В нем вызывался метод 

```c#
request.EnableRewind();
```

который находится в internal пространстве и  в новом API отсутствует. что приводит к ошибке, которую можно решить заменив его на метод

```c#
request.EnableBuffering();
```

## Сериализация

для сериализации раннее использовалась конструкция вида

```c#
services.AddMvcCore()
                    .AddJsonFormatters()
                    .AddJsonOptions(options => options.SerializerSettings.DateTimeZoneHandling = DateTimeZoneHandling.Utc)
```

сейчас аналогичного поведения можно достичь добавив пакет `Microsoft.AspNetCore.Mvc.NewtonsoftJson`

```c#
services.AddMvcCore()
                .AddNewtonsoftJson(settings => settings.SerializerSettings.DateTimeZoneHandling = DateTimeZoneHandling.Utc)
```

## Swagger

Для использования с .net core 3.0 4 версия swashbuckle уже не подходит. Соответственно требуется обновление на 

```bash
    <PackageReference Include="Swashbuckle.AspNetCore.Swagger" Version="5.0.0-rc4" />
    <PackageReference Include="Swashbuckle.AspNetCore.SwaggerGen" Version="5.0.0-rc4" />
```

После обновления полностью изменяется код, отвечающий за авторизацию на 

```c#
 c.SwaggerDoc("v1", new OpenApiInfo()
            {

                Version = "v1",
                Title = "Web API",
                Description = "The API",
                Contact = new OpenApiContact()
                {
                    Name = "Radiofisik",
                    Email = "mail@radiofisik.ru",
                },
                License = new OpenApiLicense()
                {
                    Name = "License",
                }
            });

            c.AddSecurityDefinition("oauth2", new OpenApiSecurityScheme
            {
                Type = SecuritySchemeType.OAuth2,
                Flows = new OpenApiOAuthFlows()
                {
                    Password = new OpenApiOAuthFlow()
                    {
                        TokenUrl = new Uri("/auth/connect/token", UriKind.Relative),
                        Scopes = new Dictionary<string, string>()
                        {
                        }
                    }
                }
            });

            c.AddSecurityRequirement(new OpenApiSecurityRequirement()
            {
                {
                    new OpenApiSecurityScheme
                    {
                        Reference = new OpenApiReference
                        {
                            Type = ReferenceType.SecurityScheme,
                            Id = "oauth2"
                        },
                    },
                    new List<string>()
                }
            });
```

Добавлять параметр в хедер через фильтр тоже стало несколько сложнее

```c#
 operation.Parameters.Add(new OpenApiParameter()
            {
                Name = "X-User-Param",
                Description = "param description",
                In = ParameterLocation.Header,
                Required = false,
                Schema = new OpenApiSchema()
                {
                    Type = "string"
                }
            });
```

## MVC

Для работы ` app.UseMvc();` теперь необходимо 

```c#
services.AddMvcCore()....   
.AddMvcOptions(options =>
                    {
                        options.EnableEndpointRouting = false;
                    })
```

## EF core

В EF core много оптимизаций из-за которых часто возникает ситуация когда запрос не может быть преобразован в SQL и при этом возникает ошибка.

Также изменилась конфигурация для 

```c#
  builder.OwnsMany(
                x => x.Posts,
                cfg =>
                {
                    cfg.HasForeignKey(x => x.BlogId);
                    cfg.HasKey(x => new {x.BlogId, x.PostId});
                    cfg.ToTable($@"{nameof(Posts)}");
                }
```

на

```
builder.OwnsMany(
                x => x.Posts,
                cfg =>
                {
                    cfg.WithOwner().HasForeignKey(x => x.BlogId);
                    cfg.HasKey(x => new {x.BlogId, x.PostId});
                    cfg.ToTable($@"{nameof(Posts)}");
                }
```

