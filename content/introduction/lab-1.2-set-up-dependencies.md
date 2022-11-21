+++
chapter = false
title = "Lab 1.2 Set up Dependencies"
weight = 8

+++
Start the "Data Science" Kernel, The kernel powers all of our notebook interactions. Click on "No Kernel" in the Upper Right

![](/images/select_kernel.png)

Select the `Data Science` Kernel

![](/images/select_data_science_kernel.png)

Confirm the Kernel is Started in Upper Right

![](/images/confirm_kernel_started.png)

> NOTE: YOU CAN NOT CONTINUE UNTIL THE KERNEL IS STARTED

Use `Shift+Enter` to run each cell of every notebook

**Follow Us On Twitter**

    %%html
    
    <a href="https://twitter.com/cfregly" class="twitter-follow-button" data-size="large" data-lang="en" data-show-count="false">Follow @cfreglya>
    <script async src="https://platform.twitter.com/widgets.js" charset="utf-8">script>

> Click This Button ^^ Above ^^

    %%html
    
    <a href="https://twitter.com/anbarth" class="twitter-follow-button" data-size="large" data-lang="en" data-show-count="false">Follow @anbartha>
    <script async src="https://platform.twitter.com/widgets.js" charset="utf-8">script>

> Click This Button ^^ Above ^^

**Star Our GitHub Repo**

    %%html
    
    <a class="github-button" href="https://github.com/data-science-on-aws/workshop" data-color-scheme="no-preference: light; light: light; dark: dark;" data-icon="octicon-star" data-size="large" data-show-count="true" aria-label="Star data-science-on-aws/workshop on GitHub">Stara>
    <script async defer src="https://buttons.github.io/buttons.js">script>

> Click This Button ^^ Above ^^Visit our Website

    %%html
    
    <iframe src="https://datascienceonaws.com" width="800px" height="600px"/>

> Click This Button ^^ Above ^^

**Shutting Down Kernel To Release Resources**

    %%html
    
    <p><b>Shutting down your kernel for this notebook to release resources.b>p>
    <button class="sm-command-button" data-commandlinker-command="kernelmenu:shutdown" style="display:none;">Shutdown Kernelbutton>
            
    <script>
    try {
        els = document.getElementsByClassName("sm-command-button");
        els[0].click();
    }
    catch(err) {
        // NoOp
    }    
    script>

> `Shift+Enter`

    %%javascript
    
    try {
        Jupyter.notebook.save_checkpoint();
        Jupyter.notebook.session.delete();
    }
    catch(err) {
        // NoOp
    }

> `Shift+Enter`