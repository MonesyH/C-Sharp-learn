# 一、 问题描述
* 问题： 在字符串中提取需要赋值的占位符子字符串，并通过和JObject字段对比进行赋值，最后把赋值占位符后的字符串覆盖原来的字符串
* 举例：
  传参：
  （1）***字符串Template***：``` "This is a template to test #{Data.many_credit_card_info:1.type} #{Data.many_credit_card_info:1.card_number}" ```
 （2） ***json***:
  ```
  {
    "Data":
        {
          "card_number": "123",
          "type": "visa",
          "inner_strings":[
              "s1",
              "s2"
          ],
        },
        {
          "card_number": "321",
          "type": "master",
          "inner_strings": [
              "s2_1",
              "s2_2"
          ],
        }
  }
  ```
  结果：
  ```"This is a template to test master 321"```

# 二、解决思路
## 1. 首先对Template作处理，区分#、{、}、需要匹配的子字符串、其他无需改变的子字符串
### 具体做法大概就是判断逐个字符是否为'#'井号，以及后面跟的是否为'{'左大括号号，是的话就把后面的字符串一直记录到'}'右大括号作为需要匹配的的子字符串。特殊情况是遇到数字、'_'、"."、":"也属于需要匹配的子字符串。其他则归类成无需改变的子串。

代码实现：
```
var next = SkipWhiteSpace(input); //input 就是template
if (!next.HasValue)
    yield break;

do
{
    if (next.Value == '#')
    {
        if (next.Location.Length >= 2 && next.Location.ToString()[1] == '{')
        {
            yield return Result.Value(VariableToken.Hash, next.Location, next.Remainder);
            next = next.Remainder.ConsumeChar();    
        }
        else
        {
            yield return Result.Value(VariableToken.Other, next.Location, next.Remainder);
            next = next.Remainder.ConsumeChar();    
        }
    }
    else if (next.Value == '{')
    {
        yield return Result.Value(VariableToken.OpenBracket, next.Location, next.Remainder);
        next = next.Remainder.ConsumeChar();
    }
    else if (char.IsLetter(next.Value) || char.IsDigit(next.Value) || next.Value is '_' or '.' or ':')
    {
        var keywordStart = next.Location;
        do
        {
            next = next.Remainder.ConsumeChar();
        } while (next.HasValue && char.IsLetter(next.Value));

        yield return Result.Value(VariableToken.Word, keywordStart, next.Location);
    }
    
    else if (next.Value == '}')
    {
        yield return Result.Value(VariableToken.CloseBracket, next.Location, next.Remainder);
        next = next.Remainder.ConsumeChar();
    }
    else
    {
        yield return Result.Value(VariableToken.Other, next.Location, next.Remainder);
        next = next.Remainder.ConsumeChar();
    }
    next = SkipWhiteSpace(next.Location);
} while (next.HasValue);
```
## 2. 把需要匹配的字符串拿出来去重，逐个处理
### 具体做法就是取出步骤一分类为需要匹配的子字符串token，把他们都解析成string
```
public static List<string> Parse(TokenList<VariableToken> tokens)
{
    var started = false;
    var variables = new List<string>();
    var tempQueue = new Queue<string>();
    
    foreach (var token in tokens)
    {
        if (token.Kind == VariableToken.Hash)
        {
            if (!started)
            {
                tempQueue.Enqueue(token.Span.ToString());
            }
            started = true;    
        }
        else if (token.Kind == VariableToken.OpenBracket)
        {
            if (started)
            {
                tempQueue.Enqueue(token.Span.ToString());
            }
        }
        else if (token.Kind == VariableToken.Word)
        {
            if (started)
            {
                tempQueue.Enqueue(token.Span.ToString());
            }
        }
        else if (token.Kind == VariableToken.CloseBracket)
        {
            if (started)
                tempQueue.Enqueue(token.Span.ToString());

            var variable = "";

            while (tempQueue.TryPeek(out _))
            {
                variable += tempQueue.Dequeue();
            }
            
            if (!string.IsNullOrEmpty(variable))
                variables.Add(variable);
            started = false;
        }
    }

    return variables;
}
```
## 3. 把每个需要匹配的子字符串和JObject中的字段做对比
### 具体做法是要把每次匹配过后的字段剔除掉，拿着剔除后的字符串进入下一次匹配。特殊情况是遇到List属性的字段需要划分为是下标属性字段，获取对应下标的元素。匹配成功的字段都换成JObject中的值。
```
foreach (var variable in variables)
{
    SanitizedProperty prev = null;

    var parsedVariable = new ParsedVariable(variable);

    for (var i = 0; i < parsedVariable.SubProperties.Count; i++)
    {
        var subProperty = parsedVariable.SubProperties[i];
        
        var matched = prev == null
            ? jObject.SelectToken($"$.{subProperty.FieldName}")
            : prev.Data?.SelectToken($"$.{subProperty.FieldName}");

        if (subProperty is NormalProperty)
        {
            if (matched != null)
            {
                subProperty.Data = matched;
            }
        }
        else if (subProperty is IndexedProperty typedProperty)
        {
            if (matched != null)
            {
                if (typedProperty.Index <= matched.ToList().Count - 1)
                    subProperty.Data = matched[typedProperty.Index];
            }
        }

        if (i == parsedVariable.SubProperties.Count - 1 && subProperty.Data != null)
        {
            template = template.Replace(variable, (string)subProperty.Data);
        }

        prev = subProperty;
    }
}
```
