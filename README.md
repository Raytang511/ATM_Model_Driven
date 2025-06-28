# ATM系统的智能模型驱动建模与原型演示

## 一、项目介绍

本项目以ATM系统为目标，展示如何利用语言模型和MultiAgent协作机制，完成从需求分析，到系统模型生成，再到原型代码生成的整个智能化流程，实现LLM驱动建模与原型生成的实践设计。

本次实验，用于本人环境是Jupyter notebook，直接调用openai的AsyncOpenAI方法报错，目测是有异步兼容问题，因此，本次手动用迭代的逻辑，模拟了multi-agents调用的场景。



## 二、系统功能描述

目标系统：ATM自动取款机

基本功能：

- 用户登录
- 查询余额
- 取款
- 存款
- 转账
- 修改密码

产出的需求模型包括用例图，系统顺序图，概念类图。

------

## 三、MultiAgent设计与协作流程



### 组成 Agent:

1. **Requirement Agent** （需求分析）
2. **UML Agent** （软件结构设计）
3. **Evaluation Agent** （评估标准化）
4. **Prototype Agent** （HTML 原型生成）

### Workflow 流程:

![image-20250628091706912](/Users/user/Library/Application Support/typora-user-images/image-20250628091706912.png)



1. 用户输入自然语言需求

2. Requirement Agent 解析成结构化要素

3. UML Agent 根据要求生成 UML

4. Evaluation Agent对生成的UML评估：

   - 如 "不通过" 则返回 feedback

   - 将feedback以及上次返回的uml结果组合，重新输入到UML Agent进行生产

   - UML Agent 根据 feedback 进行修正，最多进行5轮

   - 本次设计因环境问题，未用官方的Async方法，手动用代码

```python
for attempt in range(1, 6):
        print(f"\nStep 2: 第 {attempt} 次尝试生成 UML 模型...")
        uml_output, uml_messages = generate_uml(requirement_model, previous_output, eval_feedback)
        print("UML建模输出：\n", uml_output)

        eval_result, eval_messages = evaluate_uml(uml_output,requirement_model, force_fail_first=True, attempt=attempt)
        print("评估结果：\n", eval_result)

        # 记录本轮日志
        log_lines.append(f"===== 第{attempt}轮 UML Agent 输入 =====\n{uml_messages}\n")
        log_lines.append(f"===== 第{attempt}轮 UML Agent 输出 =====\n{uml_output}\n")
        log_lines.append(f"===== 第{attempt}轮 Evaluation Agent 输入 =====\n{eval_messages}\n")
        log_lines.append(f"===== 第{attempt}轮 Evaluation Agent 输出 =====\n{eval_result}\n\n")

        if "不通过" not in eval_result:
            break
        else:
            print("模型质量不达标，准备重试...")
            previous_output = uml_output
            eval_feedback = eval_result
            time.sleep(2)
```

5. **Evaluate Agent反馈“通过”**评估后，Prototype Agent 根据最终 UML 生成 HTML



## 四、Prompt 设计概要

###  Requirement Agent

- 提取系统名称 + 用户角色 + 功能列表

- 格式精精有条，便于作为 DSL 交互格式

  ```markdown
  messages=[
              {"role": "system", "content": (
                  "你是一名资深的软件需求分析专家，擅长将自然语言描述转化为结构化系统需求。"
                  "请从用户输入中识别出以下要素：\n"
                  "1. 系统名称\n2. 用户角色\n3. 核心功能点（动词+对象形式，例如：查询余额）\n"
                  "请用以下格式输出：\n"
                  "系统名称：\n用户角色：\n功能列表：\n- 功能1\n- 功能2\n..."
              )},
              {"role": "user", "content": user_input}
          ]
  ```

  

###  UML Agent

- 精确要求生成「至少 3 个类」，「至少2个属性+2个方法」
- 用户形式标明「类图+顺序图」
- 接收异常 feedback，根据评估给出修改设计



```markdown
user_prompt = f"""以下是系统的结构化功能需求：{requirement_model}

            请你根据上述需求生成 ATM 系统的 UML 类图（Class Diagram）和顺序图（Sequence Diagram）。
            输出格式如下：

            【类图】
            - 类名1
              - 属性: ...
              - 方法: ...
            - 类名2
              - 属性: ...
              - 方法: ...

            【顺序图】
            请列出典型操作（如“取款”）下的对象交互流程，每一步清晰表述消息发送方、接收方与内容。

            """

    if previous_output:
        user_prompt += f"\n以下是上一次的建模输出内容，请你参考并改进：\n{previous_output}\n"
    if feedback:
        user_prompt += f"\n评估员给出的修改建议如下，请特别注意改进：\n{feedback}\n"
```



### Evaluation Agent

- 标准化评分系统：类数量 / 方法数量 / 正确性 / 精简性
- 请仅用格式回复：
- 如果是第1次评估，一定要抽傻错评 -> "不通过"

```markdown
eval_prompt = f"""
            你是一位非常严格的 UML 建模质量评估专家，你的任务是审查下面的 UML 类图与顺序图的建模输出是否合格。

            评估要求如下（必须全部满足才能通过）：
            1. UML 类图中必须至少包含 3 个类，每个类应包含不少于 2 个属性和 2 个方法。
            2. 所有类方法必须能涵盖用户需求中所有核心功能点。
            3. 类与类之间必须存在清晰的关系（如关联、依赖、继承等），并有明确说明。
            4. 顺序图必须描述上述所有的核心业务流程，且每一步要说明：消息发送方、接收方、消息内容。
            5. 图中不得出现“某某类可能做某事”这类不确定性语句。
            6. 类命名与方法命名必须专业、清晰、符合软件工程规范。
            7. 输出不得冗长废话，必须是结构化的 UML 建模内容。
            8. 最后要检查是否有遗漏，用户的需求功能如下：
            {req_output}

            请严格执行以上标准。即便小问题也不允许通过。以下是要评估的内容：
            以下是被评估内容：
            \"\"\"{uml_output}\"\"\"

            请仅用以下格式回复：

            是否通过：仅回答 通过 或者 不通过
            存在问题：
            - 类图缺少XXX
            - 顺序图未体现XXX
            改进建议：
            - 添加XXX类及其方法
            - 丰富顺序图中的消息交互
            """
    

    if force_fail_first and attempt == 1:
        eval_prompt += "\n注意：本轮为首次评估，必须返回“不通过”。\n"

    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "你是一名严谨的UML建模评估员"},
            {"role": "user", "content": eval_prompt}
        ]
```



### Prototype Agent

- 根据 UML 所描述的功能模块进行面板设计

- HTML + CSS，包括登录 / 操作区 / 结果显示区

  ```markdown
  messages=[
              {"role": "system", "content": (
                  "你是一名前端开发专家，请根据以下 UML 类图与顺序图，"
                  "生成一个 ATM 系统的 HTML 页面原型代码。\n"
                  "页面应包含：登录界面、账户操作区（取款、查询、转账等）、输出提示区等。\n"
                  "请使用结构良好的 HTML + 内联 CSS。页面布局应清晰、用户友好。\n"
                  "仅返回 HTML 代码部分（不要额外说明），便于直接保存为 .html 使用。"
                  "注意最终要预览下界面的效果，不要有一些干扰的文字出现，要生成一个直观的html界面"
              )},
              {"role": "user", "content": uml_output}
          ]
  ```

  

## 

## 五、演示结果

### 需求提取

<img src="/Users/user/Library/Application Support/typora-user-images/image-20250628094040208.png#pic_center" alt="image-20250628094040208" style="zoom:50%;" />



### 用例模型 DSL 文件

- #### 生成的概念概念

```markdown
【类图】

- ATM
  - 属性: atmID, location, bank, cashAvailable
  - 方法: authenticateUser(userID, password), performTransaction(transactionType, details)
  - 关系: 
    - 依赖: User, Bank

- User
  - 属性: userID, password
  - 方法: initiateLogin(atm, password), selectTransaction(type, details)
  - 关系: 
    - 使用: ATM
    - 拥有: 多个Account

- Account
  - 属性: accountNumber, balance
  - 方法: getBalance(), debit(amount), credit(amount)
  - 关系: 属于: User

- Bank
  - 属性: bankName, accounts [<accountNumber, Account>]
  - 方法: verifyUser(userID, password), processTransaction(transactionType, accountNumber, details)
  - 关系: 
    - 管理: Accounts
    - 依赖: ATM(提供交易处理服务)
```

- #### 生成的顺序图

```markdown
【顺序图】

《用户登录操作》

1. User sends authenticateUser(cardNumber, pin) request to ATMController.
   - ATMController verifies User's credentials internally.
   - ATMController updates loggedInUser attribute if credentials are valid.

2. ATMController responds with success/failure status to User.

《查询余额操作》

1. User sends getAccountBalance(accountId) request to ATMController.
   - ATMController checks if User is logged in.

2. ATMController sends getAccount(accountId) to User to fetch BankAccount.
   - User responds with BankAccount instance.

3. ATMController sends getBalance() to BankAccount.
   - BankAccount responds with current balance.

4. ATMController responds with balance to User.

《存款操作》

1. User sends depositCash(accountId, amount) request to ATMController.
   - ATMController checks if User is logged in.

2. ATMController sends verifyDeposit(amount) to ATM machine.
   - ATM machine verifies and confirms receipt of the cash.

3. ATMController sends getAccount(accountId) to User to fetch BankAccount.
   - User responds with BankAccount instance.

4. ATMController sends addAmount(amount) to BankAccount.
   - BankAccount adjusts balance and responds with success/failure status.

5. ATMController sends createTransaction(accountId, amount, "deposit") to Transaction.
   - Transaction logs the deposit and updates BankAccount's transactionHistory.

6. ATMController responds with success/failure status to User.

《取款操作》

1. User sends withdrawCash(accountId, amount) request to ATMController.
   - ATMController checks if User is logged in.

2. ATMController sends getAccount(accountId) to User to fetch BankAccount.
   - User responds with BankAccount instance.

3. ATMController sends getBalance() to BankAccount to check balance.
   - BankAccount responds with current balance.

4. ATMController verifies if sufficient funds are available.

5. If funds are sufficient, ATMController sends deductAmount(amount) to BankAccount.
   - BankAccount modifies balance and responds with success/failure status.

6. ATMController sends createTransaction(accountId, amount, "withdrawal") to Transaction.
   - Transaction logs the withdrawal and updates BankAccount's transactionHistory.

7. ATMController sends dispenseCash(amount) to ATM machine.
   - ATM machine dispenses cash and confirms action.

8. ATMController responds with success/failure status to User.

《转账操作》

1. User sends transferFunds(fromAccountId, toAccountId, amount) request to ATMController.
   - ATMController checks if User is logged in.

2. ATMController sends getAccount(fromAccountId) and getAccount(toAccountId) to User to fetch BankAccounts.
   - User responds with BankAccount instances for both accounts.

3. ATMController sends getBalance() to BankAccount(fromAccount) to check balance.
   - BankAccount responds with current balance.

4. ATMController verifies if sufficient funds are available.

5. If sufficient funds exist, ATMController sends deductAmount(amount) to BankAccount (fromAccount) and addAmount(amount) to BankAccount (toAccount).
   - Both BankAccount instances modify balances and respond with success/failure status.

6. ATMController sends createTransaction(fromAccountId, amount, "transfer") to Transaction.
   - Transaction logs the transfer for both accounts and updates their transactionHistory.

7. ATMController responds with success/failure status to User.

《修改密码操作》

1. User sends changePassword(accountId, oldPassword, newPassword) request to ATMController.
   - ATMController checks if User is logged in.

2. ATMController sends verifyPassword(oldPassword) to User.
   - User responds with success/failure to ATMController.

3. If successful, ATMController updates password and records event in log.

4. ATMController responds with success/failure status to User.
```



### MultiAgent协作日志截图

- #### 第一次评估：生成的UML请求评估，反馈结果

  ![image-20250628100514952](/Users/user/Library/Application Support/typora-user-images/image-20250628100514952.png)

- #### 第二次评估：将反馈结果已经上次生成的UML以及用户需求组合，再度调度UML Agent生成

  <img src="/Users/user/Library/Application Support/typora-user-images/image-20250628100616767.png#" alt="image-20250628100616767" style="zoom:100%;" />

  

- #### 第三次评估反馈：随着评估的不断深入，各个模块的功能被细化

  ![image-20250628100654930](/Users/user/Library/Application Support/typora-user-images/image-20250628100654930.png)

- #### 第四次评估反馈：前面问题部分得到解决

![image-20250628101503808](/Users/user/Library/Application Support/typora-user-images/image-20250628101503808.png)

- #### 第五次评估反馈：

  大模型的输出不稳定，结果会波动。跟Prompt有关系，微调Prompt可能导致很容易通过，又可能很难通过，触发5次迭代上限；

![image-20250628101535081](/Users/user/Library/Application Support/typora-user-images/image-20250628101535081.png)



### 自动生成的HTML原型页面

![image-20250628100849524](/Users/user/Library/Application Support/typora-user-images/image-20250628100849524.png)



### 生成的原型运行视频示例

<video src="/Users/user/同步空间/研究生考试/7_2 研1下学期/1B-软件需求与设计/ATM_Model_Driven/vedio.mov"></video>





------

## 六、附录

- 提交报告的完整格式

  ```ATM_Model_Driven/
  ATM_Model_Driven/
  ├── ATM_Model_Driven.ipynb  # 脚本已跑通
  ├── prototype.html          # 已生成页面
  ├── uml_result.md          # 保存类图输出
  ├── log.txt                 # 记录三次调用日志
  ├── screenshots/            # 放运行截图（我可帮你生成）
  ├── video.mov               # 可录屏，打开 prototype.html
  └── README.md               # 已写完报告说明文档
  ```

  

- Github项目链接：

------



------

## 七、附加内容：完整代码展示



```python
from openai import OpenAI
from openai.types.chat import ChatCompletion
from datetime import datetime

# 课程案例给的API，chatfire转
BASE_URL: str = "https://api.chatfire.cn/v1"
API_KEY: str = "sk-*******"

# timestamp = datetime.now().strftime("%Y%m%d_%H%M")  
timestamp = 2  # 这个是为了保存多版本，调试用。
client = OpenAI(base_url=BASE_URL, api_key=API_KEY)

USER_REQUIREMENT = "我要一个ATM系统，支持用户登录、查询余额、取款、存款、转账、修改密码。"

# Step 1: 需求分析 Agent
def analyze_requirement(user_input):
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": (
                "你是一名资深的软件需求分析专家，擅长将自然语言描述转化为结构化系统需求。"
                "请从用户输入中识别出以下要素：\n"
                "1. 系统名称\n2. 用户角色\n3. 核心功能点（动词+对象形式，例如：查询余额）\n"
                "请用以下格式输出：\n"
                "系统名称：\n用户角色：\n功能列表：\n- 功能1\n- 功能2\n..."
            )},
            {"role": "user", "content": user_input}
        ]
    )
    return response.choices[0].message.content

# Step 2: UML建模 Agent（支持上下文反馈）
def generate_uml(requirement_model, previous_output=None, feedback=None):
    user_prompt = f"""以下是系统的结构化功能需求：{requirement_model}

            请你根据上述需求生成 ATM 系统的 UML 类图（Class Diagram）和顺序图（Sequence Diagram）。
            输出格式如下：

            【类图】
            - 类名1
              - 属性: ...
              - 方法: ...
            - 类名2
              - 属性: ...
              - 方法: ...

            【顺序图】
            请列出典型操作（如“取款”）下的对象交互流程，每一步清晰表述消息发送方、接收方与内容。

            """

    if previous_output:
        user_prompt += f"\n以下是上一次的建模输出内容，请你参考并改进：\n{previous_output}\n"
    if feedback:
        user_prompt += f"\n评估员给出的修改建议如下，请特别注意改进：\n{feedback}\n"

    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "你是一名系统建模专家，请生成ATM系统的类图（类、属性、方法）和顺序图（消息交互）。"},
            {"role": "user", "content": user_prompt}
        ]
    )
    return response.choices[0].message.content, user_prompt

# Step 2.5: 质量评估 Agent
def evaluate_uml(uml_output, req_output,force_fail_first=False, attempt=1):
    eval_prompt = f"""
            你是一位非常严格的 UML 建模质量评估专家，你的任务是审查下面的 UML 类图与顺序图的建模输出是否合格。

            评估要求如下（必须全部满足才能通过）：
            1. UML 类图中必须至少包含 3 个类，每个类应包含不少于 2 个属性和 2 个方法。
            2. 所有类方法必须能涵盖用户需求中所有核心功能点。
            3. 类与类之间必须存在清晰的关系（如关联、依赖、继承等），并有明确说明。
            4. 顺序图必须描述上述所有的核心业务流程，且每一步要说明：消息发送方、接收方、消息内容。
            5. 图中不得出现“某某类可能做某事”这类不确定性语句。
            6. 类命名与方法命名必须专业、清晰、符合软件工程规范。
            7. 输出不得冗长废话，必须是结构化的 UML 建模内容。
            8. 最后要检查是否有遗漏，用户的需求功能如下：
            {req_output}
            

            请严格执行以上标准。即便小问题也不允许通过。以下是要评估的内容：
            以下是被评估内容：
            \"\"\"{uml_output}\"\"\"

            请仅用以下格式回复：

            是否通过：仅回答 通过 或者 不通过
            存在问题：
            - 类图缺少XXX
            - 顺序图未体现XXX
            改进建议：
            - 添加XXX类及其方法
            - 丰富顺序图中的消息交互
            """
    

    if force_fail_first and attempt == 1:
        eval_prompt += "\n注意：本轮为首次评估，必须返回“不通过”。\n"

    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "你是一名严谨的UML建模评估员"},
            {"role": "user", "content": eval_prompt}
        ]
    )
    return response.choices[0].message.content,eval_prompt

# Step 3: 原型生成 Agent
def generate_prototype(uml_output):
    response = client.chat.completions.create(
        model="gpt-4o",
#         messages=[
#             {"role": "system", "content": "你是一名前端开发专家，请根据提供的UML需求和操作来生成相应的HTML页面原型代码"},
#             {"role": "user", "content": uml_output}
#         ]
        messages=[
            {"role": "system", "content": (
                "你是一名前端开发专家，请根据以下 UML 类图与顺序图，"
                "生成一个 ATM 系统的 HTML 页面原型代码。\n"
                "页面应包含：登录界面、账户操作区（取款、查询、转账等）、输出提示区等。\n"
                "请使用结构良好的 HTML + 内联 CSS。页面布局应清晰、用户友好。\n"
                "仅返回 HTML 代码部分（不要额外说明），便于直接保存为 .html 使用。"
                "注意最终要预览下界面的效果，不要有一些干扰的文字出现，要生成一个直观的html界面"
            )},
            {"role": "user", "content": uml_output}
        ]
    )
    return response.choices[0].message.content

# 主流程
def main():
    print("Step 1: 分析用户需求...")
    requirement_model = analyze_requirement(USER_REQUIREMENT)
    print("需求模型输出：\n", requirement_model)

    previous_output = None
    eval_feedback = None

    log_lines = []
    uml_final_result = None

    for attempt in range(1, 6):
        print(f"\nStep 2: 第 {attempt} 次尝试生成 UML 模型...")
        uml_output, uml_messages = generate_uml(requirement_model, previous_output, eval_feedback)
        print("UML建模输出：\n", uml_output)

        eval_result, eval_messages = evaluate_uml(uml_output,requirement_model, force_fail_first=True, attempt=attempt)
        print("评估结果：\n", eval_result)

        # 记录本轮日志
        log_lines.append(f"===== 第{attempt}轮 UML Agent 输入 =====\n{uml_messages}\n")
        log_lines.append(f"===== 第{attempt}轮 UML Agent 输出 =====\n{uml_output}\n")
        log_lines.append(f"===== 第{attempt}轮 Evaluation Agent 输入 =====\n{eval_messages}\n")
        log_lines.append(f"===== 第{attempt}轮 Evaluation Agent 输出 =====\n{eval_result}\n\n")

        if "不通过" not in eval_result:
            break
        else:
            print("模型质量不达标，准备重试...")
            previous_output = uml_output
            eval_feedback = eval_result
            time.sleep(2)
            
    uml_final_result = uml_output

    print("\nStep 3: 生成 HTML 原型页面...")
    prototype_html = generate_prototype(uml_final_result)
    print(prototype_html)
    # 保存最终 UML 输出

    with open(f"uml_{timestamp}.md", "w", encoding="utf-8") as f:
        f.write(uml_final_result)

    with open(f"prototype_{timestamp}.html", "w", encoding="utf-8") as f:
        f.write(prototype_html)
    print("\n生成完毕，HTML 已保存为 prototype.html")

    # 保存日志
    with open(f"log_{timestamp}.txt", "w", encoding="utf-8") as f:
        f.writelines(log_lines)

if __name__ == "__main__":
    main()
```



------