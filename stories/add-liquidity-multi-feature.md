<!--
Provide a general summary of the issue in the title above and use relevant 
fields below to define the problem.
-->

#### User Story
<!--
- Audience or user can include a person or system, i.e. dev, user, api.
- An action or task this issue will accomplish.
- What is the desired outcome or goal?

NOTE: Feel free to replace this with a general description if a user story doesn't make sense, but
be willing to defend your choice to exclude a user story.
-->
- As an <audience/user>: liquidity provider
- I want to <action/task>: add liquidity from all assets
- so that <outcome/goal/benefit>: I can get pool tokens with all of the included assets

#### Type
<!--
- Select a type of issue
-->
- [X] Enhancement
- [ ] Maintenance
- [ ] Refactor

#### Description
<!--
- Describe the problem and why this task is needed.
-->

The first box on the right column will be a box allowing the user to add or withdraw liquidity. For the purpose of this story, we will be focusing on adding liquidity from multiple assets. The top line of the the box will contain the words `Add Liquidity` in extra large bold text. The second line will contain a toggle button with the words `Single Asset` on the left and `Multi Asset` on the right. This toggle button will be set to `Single Asset` by default. This story deals exclusively with the scenario where `Multi Asset` has been selected. The third line will contain the text `Multi Asset entry enables you to mint [lowercase-sybmol] according to it's allocation`.

Directly below the third line will be a button-tab style select with `Add Liquidity` on the left and `Withdraw` on the right. This story deals exclusively with the scenario where `Add Liquidity` is selected.

Directly below the add and remove liquidity button-tabs will be a white box allowing the liquidity provider to input the amount and single asset they intend to use to mint the pool token. On the left hand side the first line will read `From (Balance: [amount])` in small text. The amount will consist of 6 total digits (ex. 2673.09 or 3.67320). The second line on the left hand side will be an input box allowing the user to type the exact amount of tokens to put towards minting. This input will default to `1.00000000` and not have any border or background. It's value text will be extra large. On the first line of the right hand side will be 4 buttons with black backgrounds and white text, 25%, 50%, 75%, & 100%. These buttons will be calculated with a 2% decrease (so 100% is actually 98%) as to allow for any short term slippage in price. When clicking a button, it will chnage the input to match the appropriate amount. On the second row of the right hand side will be an unlock button (not shown if an adequate amount is already unlocked) with a transparent background and 1 pixel solid black border. Following this will be a grey button which defaultly shows the asset with the highest weight (logo then sybmol) followed by a downward facing carrot. When the unlock button is click, a transaction will be generated to approve the maximum amount of the token selected. When the token button is clicked, a modal will pop up allowing for the selection of another one of the pool assets [See Issue #TBD].

A downward facing arrow leads to a list of required assets in white boxes with 4 columns. The first column will contain the percentage required (4 digits) followed by the token logo and symbol. A 1px black line border will separate the first and second columns. The second column will contain the amount of tokens required (9 digits total) in bold, rounding up. The third column will contain the balance available minus the amount required (is this correct??). If the number is negative, it will be red, positive numbers will be green. The fourth column will contain a BUY button (black with white letters) if there is not enough token available (where does this link?), an unlock button (white background, black text and black 1px border) if the token is not yet approved, or ?????.

Directly below will be a button titled `Add Liquidity` which generates the transaction required. If any of the token amounts request are higher than the available amount of the tokens (... ??????). If any of the tokens are not yet unlocked, an approve transaction is generated for each before the enter pool function is called. Once the tokens are approved, the enter pool function is called.

#### Definition of Done
<!--
- How do you know when this issue is completed?
- List acceptance criteria, bullet points are always preferred.
-->

- [ ] `Add Liquidity` title
- [ ] `Single Asset` `Multi Asset` toggle with `Single Asset` selected
- [ ] `Single Asset entry enables you to mint [lowercase-asset] with only one token`
- [ ] `Add Liquidity` `Withdraw` button tabs, `Add Liquidity` selected
- [ ] Input box with free input, as well as 25% 50% 75% and 100% buttons and token select button
- [ ] Down arrow
- [ ] List of pool tokens required
- [ ] Add Liquidity button

#### Attach files or screenshots if it's helpful for this issue.

![simple page](https://piedao-productpage-improvements.netlify.app/img/page08.png)