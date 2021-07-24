---
title: Разбор выражений
description: вычисление формул
---

Допустим что надо создать парсер для вычисления значений математических выражений, который понимает основные арифметические операции +-*/ умеет учитывать их порядок и изменять его с помощью скобок. Также необходимо реализовать возможность расширения кастомными функциями. Результатом вычисления выражения будет тип double.

Вычисление выражения делится на два этапа:

- Парсинг выражения в дерево классов выражений
- Вычисления значения полученного выражения вызовом метода корневого элемента выражения

Создадим абстрактный класс выражения

```c#
public abstract class BaseExpression
{
    public abstract double Compute(Dictionary<string, object> context);
}
```

Простейшей реализацией этого класса будет класс константы

```c#
public class ConstExpression : BaseExpression
{
    public double Value { get; }

    public ConstExpression(double value)
    {
        Value = value;
    }
    
    public override double Compute(Dictionary<string, object> context)
    {
        return Value;
    }

    public override string ToString() => $"Const({Value})";
}
```

Для вычислений с переменными из контекста класс переменной

```c#
public class VariableExpression : BaseExpression
{
    public string Name { get; }

    public VariableExpression(string name)
    {
        Name = name;
    }

    public override double Compute(Dictionary<string, object> context)
    {
        return (double) context[Name];
    }

    public override string ToString() => $"Var({Name})";
}
```

В арифметике много бинарных операций например выражение 3+2 имеет левую часть 3, оператор + и правую часть 2. для описания таких операций введем класс

```c#
public abstract class BaseBinaryExpression : BaseExpression
{
    public BaseExpression Left { get; }

    public BaseExpression Right { get; }

    protected BaseBinaryExpression(BaseExpression left, BaseExpression right)
    {
        Left = left;
        Right = right;
    }
}
```

С помощью этого класса можно описать выражения +-*/, например сложение

```c#
public class PlusExpression : BaseBinaryExpression
{
    public PlusExpression(BaseExpression left, BaseExpression right) : base(left, right)
    {
    }

    public override double Compute(Dictionary<string, object> context)
    {
        return Left.Compute(context) + Right.Compute(context);
    }

    public override string ToString() => $"{Left} + {Right}";
}
```

и скобок, по сути выражение скобок тут формально, поскольку порядок операций определяется структурой дерева, но они нужны для того чтобы обратно сгенерированная строка выражения была корректна

```c#
public class ParenthesisExpression : BaseExpression
{
    public BaseExpression Expression { get; }

    public ParenthesisExpression(BaseExpression expression)
    {
        Expression = expression;
    }

    public override double Compute(Dictionary<string, object> context)
    {
        return Expression?.Compute(context) ?? 0.0;
    }

    public override string ToString() => $"({Expression})";
}
```

Имея эту структуру классов можно уже что-нибудь посчитать

```c#
//2*(2+3)
var exp = new MultiplyExpression( new ConstExpression(2),
                                 new ParenthesisExpression(new PlusExpression(new ConstExpression(2), new ConstExpression(3))));
var result = exp.Compute(new Dictionary<string, object>());
```

Осталось самое сложное - написать парсер, который преобразует строку в класс выражения `BaseExpression Parse(string input);` Для начала надо описать формальную грамматику

```c#
expr : plusminus* EOF ;
plusminus: multdiv ( ( '+' | '-' ) multdiv )* ;
multdiv : factor ( ( '*' | '/' ) factor )* ; // multdiv это factor дальше * или / и другой factor, при этом оператор и фактор могут повторяться бесконечно
factor : NUMBER | '(' expr ')' ; // factor это NUMBER или expr в скобках
```

 Разобьем задачу на два этапа, найдем в строке лексемы

```c#
private LexemeBuffer ParseLexemes(ReadOnlySpan<char> input)
{
    var lexemes = new LexemeBuffer();
    var current = 0;
    while (current < input.Length)
    {
        switch (input[current])
        {
            case '(':
                lexemes.Add(new Lexeme(LexemeType.LeftBracket, input[current]));
                current++;
                break;
            case ')':
                lexemes.Add(new Lexeme(LexemeType.RightBracket, input[current]));
                current++;
                break;
            case '+':
                lexemes.Add(new Lexeme(LexemeType.Plus, input[current]));
                current++;
                break;
            case '-':
                lexemes.Add(new Lexeme(LexemeType.Minus, input[current]));
                current++;
                break;
            case '*':
                lexemes.Add(new Lexeme(LexemeType.Multiply, input[current]));
                current++;
                break;
            case '/':
                lexemes.Add(new Lexeme(LexemeType.Divide, input[current]));
                current++;
                break;
            default:
                if (Char.IsDigit(input[current]))
                {
                    var start = current;
                    while (current < input.Length && char.IsDigit(input[current]) || input[current] == '.' || input[current] == ',')
                    {
                        current++;
                    }

                    lexemes.Add(new Lexeme(LexemeType.Number, input.Slice(start, current - start).ToString()));
                }

                break;
        }
    }

    return lexemes;
}

```

Теперь начнем реализацию метода рекуррентного спуска. Начнем снизу вверх, согласно правилам factor это NUMBER или expr в скобках, expr умеет парсить метод, который реализуем потом, из Number же создаем `ConstExpression(double.Parse(lexeme.Value))`

```c#
private BaseExpression Factor(LexemeBuffer lexemes)
{
    var lexeme = lexemes.Next();
    switch (lexeme.Type)
    {
        case LexemeType.Number:
            return new ConstExpression(double.Parse(lexeme.Value));
        case LexemeType.LeftBracket:
            var result = new ParenthesisExpression(Expr(lexemes));
            lexeme = lexemes.Next();
            if (lexeme.Type != LexemeType.RightBracket)
            {
                throw new Exception($") expected at position {lexemes.Position}");
            }

            return result;
        default:
            throw new Exception($"syntax error at {lexemes.Position}");
    }
}
```

Аналогично, поднимаясь вверх по правилам реализуем 

```c#
private BaseExpression Expr(LexemeBuffer lexemes)
{
    var lexeme = lexemes.Next();
    if (lexeme.Type == LexemeType.End)
    {
        throw new Exception("expression is empy");
    }

    lexemes.Back();
    return PlusMinus(lexemes);
}

private BaseExpression PlusMinus(LexemeBuffer lexemes)
{
    var result = Multiply(lexemes);
    while (true)
    {
        var lexeme = lexemes.Next();
        switch (lexeme.Type)
        {
            case LexemeType.Plus:
                result = new PlusExpression(result, Multiply(lexemes));
                break;
            case LexemeType.Minus:
                result = new MinusExpression(result, Multiply(lexemes));
                break;
            default:
                lexemes.Back();
                return result;
        }
    }
}

private BaseExpression Multiply(LexemeBuffer lexemes)
{
    var result = Factor(lexemes);
    while (true)
    {
        var lexeme = lexemes.Next();
        switch (lexeme.Type)
        {
            case LexemeType.Multiply:
                result = new MultiplyExpression(result, Factor(lexemes));
                break;
            case LexemeType.Divide:
                result = new DivideExpression(result, Factor(lexemes));
                break;
            default:
                lexemes.Back();
                return result;
        }
    }
}
```

Осталось реализовать вычисление функций

```
expr : plusminus* EOF ;
plusminus: multdiv ( ( '+' | '-' ) multdiv )* ;
multdiv : factor ( ( '*' | '/' ) factor )* ; // multdiv это factor дальше * или / и другой factor, при этом оператор и фактор могут повторяться бесконечно
func : name(expr(; expr)*)
factor : NUMBER | '(' expr ')' | func ; // factor это NUMBER или expr в скобках
```

Для реализации этой функции в лексер добавим

```c#
else if(Char.IsLetter(input[current]))
{
    var start = current;
    while (current < input.Length && char.IsLetter(input[current]))
    {
        current++;
    }

    lexemes.Add(new Lexeme(LexemeType.Func, input.Slice(start, current - start).ToString()));
}
```

в парсер фактора 

```c#
    case LexemeType.Func:
               lexemes.Back();
               return Func(lexemes);
```

для функций с одним параметром можно написать так

```c#
 private BaseExpression Func(LexemeBuffer lexemes)
      {
         var lexeme = lexemes.Next();
         var funcName = lexeme.Value;

         lexeme = lexemes.Next();
         if (lexeme.Type != LexemeType.LeftBracket)
         {
            throw new Exception($"( expected at position {lexemes.Position}");
         }

         var result = new FuncExpression(funcName, new List<BaseExpression>(){Expr(lexemes)});

         lexeme = lexemes.Next();
         if (lexeme.Type != LexemeType.RightBracket)
         {
            throw new Exception($") expected at position {lexemes.Position}");
         }
         return result;
      }
```

где само выражение позволит вычислять корни

```c#
 public class FuncExpression: BaseExpression
   {
      private readonly List<BaseExpression> _params;
      private string _name;

      public FuncExpression(string name, List<BaseExpression> @params)
      {
         _params = @params;
         _name = name;
      }

      public override double Compute(Dictionary<string, object> context)
      {
         if (_name == "sqrt")
         {
            return  Math.Sqrt(_params.First().Compute(context));
         }
         else
         {
            throw new Exception($"function with name {_name} was not found");
         }
      
      }
   }
```

данный код уже нормально справляется в выражениями вида 2*sqrt(25), но бывают функции со многими аргументами введем сепаратор, допустим ; . Добавим его в лексер. А парсер функции научим разбирать много параметров

```c#
 private BaseExpression Func(LexemeBuffer lexemes)
      {
         var lexeme = lexemes.Next();
         var funcName = lexeme.Value;

         lexeme = lexemes.Next();
         if (lexeme.Type != LexemeType.LeftBracket)
         {
            throw new Exception($"( expected at position {lexemes.Position}");
         }
         var funcParams = new List<BaseExpression>();
         while (true)
         {
            funcParams.Add(Expr(lexemes));
            lexeme = lexemes.Next();
            if (lexeme.Type == LexemeType.ParamSeparator)
            {
               continue;
            }
            if (lexeme.Type != LexemeType.RightBracket)
            {
               throw new Exception($") expected at position {lexemes.Position}");
            }
            return  new FuncExpression(funcName, funcParams);
         }

      }
```

дописав само вычисление функции

```c#
 if (_name == "min")
         {
            return  Math.Min(_params[0].Compute(context), _params[1].Compute(context));
         }
```

получим код, успешно вычисляющий выражение вида `"2*min(25;10)"`

> Получившийся код https://github.com/Radiofisik/ExpressionParser

Использованные источники:

- https://www.youtube.com/watch?v=iLnNqqom5KY

- https://www.youtube.com/watch?v=UIf-QQjsBuY&list=PLeQDJtBkrIiT0TMQ3muv3zvNdsmBZFOR1&index=4

- https://github.com/Arhiser/java_tutorials/blob/master/src/ru/arhiser/parser/Main.java