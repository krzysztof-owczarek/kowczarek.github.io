## Alias-like functionality in subshells / external subshells (ex. Docker, K8s, etc.)

**Aliases** or bash functions are mostly useful for frequently used commands, that are far too long or complicated to type in every time we need to execute them.

They work perfectly well **locally**, but are not very useful for the **(external) subshells**, which are quite popular in the contenerized world and in the cloud.

Recently, I have tried to find out how I could utilise aliases remotely and save myself the trouble of copy-pasting long commands from the text file over and over again or worse. Type them in every time I need to 😨

### The solution with a big disadvantage - Warp Terminal (state for: 12th May 2024)

Some of you probably know **Warp Terminal** already, as it is available on the market for a noticable period of time now.

It is advertised as a next-gen terminal and they kind of deliver their promisses, compared to the well known, classic terminals like iTerm, at least at their vanilla flavoured form.

... those of you, who already know Warp, probably also know what the big disadvantage is,

... for those of you who have never checked it out, here is a very quick one-liner summary to save you the trouble: 


🟥 **Warp Terminal requires you to log in to use it.** 🟥


```
A terminal. Requires. Logging in.

Nobody likes it, but they keep doing it - why?

There are multiple opinions on that matter.
```

### Nevertheless. For those of you, who are still curious...

Among all features that Warp Terminal offers, there is one called [*workflows*](https://docs.warp.dev/features/warp-drive/workflows) - [here](https://docs.warp.dev/features/entry/yaml-workflows#creating-custom-workflows) you can find more about creating them yourself.

Long story short, you can define special sets of (parametrized) commands in `.yaml` files, stored locally in a special directory.

Workflows are easily searchable and quickly accessible through the Warp Terminal.

### How is it different from having aliases defined in my local `.rc` file?

Warp Terminal allows you to [*warpify*](https://docs.warp.dev/features/subshells#how-to-warpify-the-subshell) any bash subshell that has been opened in their terminal.


<img width="825" alt="Screenshot 2024-05-12 at 15 04 39" src="https://github.com/krzysztof-owczarek/kowczarek.github.io/assets/38732989/8b4a8e7e-aaf8-452c-aa1b-493ed42a47aa">

and some *warpifying*... 🪄

<img width="1002" alt="Screenshot 2024-05-12 at 15 05 00" src="https://github.com/krzysztof-owczarek/kowczarek.github.io/assets/38732989/664d1cad-834f-4e77-beb3-87b0b3c3dffd">

### Quick search and workflow execution

Right now you can easily use any of the built-in workloads, as well as the ones you create yourself.

You can open a quicksearch for workflows by using a keyboard shortcut (`shift+control+r` on mac) and find any workflow you need.

<img width="994" alt="image" src="https://github.com/krzysztof-owczarek/kowczarek.github.io/assets/38732989/916fbce0-6c81-49cd-8baf-9b3550603f43">

Workloads on the screenshot are default workloads available in the Warp Terminal.

### Important disclaimer

I am not connected to the company behind Warp Terminal in any way - it just happens to be a tool with a potential solution to the problem.

Given Warp Terminal log in [policy](https://docs.warp.dev/getting-started/privacy) - you have to make the decision about using it yourself.

