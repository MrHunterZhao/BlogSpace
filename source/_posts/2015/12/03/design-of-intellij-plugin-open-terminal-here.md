---
title: 关于IntelliJ IDEA插件Open Terminal Here的设计思路
date: 2015-12-03 10:31:00
categories: IntelliJ IDEA
tags:
    - IntelliJ IDEA
    - 插件
---
![IntelliJ Plugin Repo](/images/post/2015/12/03/intellij-plugin-banner.jpg)

Open Terminal Here是一款用于在终端中直接打开文件所在目录的IntelliJ插件，于2015年12月3日发布至[IntelliJ IDEA官方插件库](https://plugins.jetbrains.com/plugin/8081-open-terminal-here)，目前有3K+次下载，源代码托管于[GitHub](https://github.com/MrHunterZhao/OpenTerminalHere)。


<!-- more -->

# 痛点场景
一直使用IntelliJ IDEA作为IDE，深爱它的智能，正如其名。然而，再完美的东西也有不完美的地方。

作为命令行重度用户自然免不了在命令行进行操作，这种情况下就不得不逐级cd到某个工程文件所在目录，如果目录层级特别深，那酸爽不可思议。

那么，问题来了，如何从命令行中一步切换到文件所在目录？答案就是：给IntelliJ写插件。


# 实现思路
IntelliJ IDEA开放了一套完整的OpenAPI，通过该OpenAPI几乎可以完成对IDEA各个组件的操作及扩展，非常方便。

思路大致为：

1. 首先通过OpenAPI获取工程文件的`VirtualFile`对象，VirtualFile是对文件系统的封装，包含了工程文件几乎所有信息，包括所在系统目录。
2. 将文件系统目录封装成对应OS的自定义Command对象
3. 通过`Runtime.getRuntime().exec(command)`调用系统命令行并切换至目标目录


# 代码实现
## 通过OpenAPI获取工程文件所在目录


```java
/**
 * Author: Hunter Zhao
 * Date:   12/2/15 18:31
 * Description: tool for file operation
 */
public class FileUtil {

    /**
     * determine the VirtualFile which selected in Project view
     * @param project
     * @return
     */
    public static VirtualFile getSelectedProjectFile(@NotNull Project project) {

        AbstractProjectViewPane currentProjectViewPane = ProjectView.getInstance(project).getCurrentProjectViewPane();
        if (currentProjectViewPane == null) {
            return null;
        }
        DefaultMutableTreeNode node = currentProjectViewPane.getSelectedNode();
        if (node == null) {
            return null;
        }
        Object selected = null;
        Object userObject = node.getUserObject();
        if (userObject instanceof AbstractTreeNode) {
            selected =  ((AbstractTreeNode)userObject).getValue();
        }
        else  if (userObject instanceof NodeDescriptor) {
            selected =  ((NodeDescriptor)userObject).getElement();
        }

        if (selected == null) {
            return null;
        }

        VirtualFile vf = null;
        if (selected instanceof PsiDirectory) {
            vf = ((PsiDirectory)selected).getVirtualFile();
        }
        else if (selected instanceof PsiElement) {
            vf = ((PsiElement) selected).getContainingFile().getVirtualFile().getParent();
        }
        else {
            // ignored
        }

        return vf;
    }
}

```



## 封装成Command对象

1. Command对象

```java
public class Command {

    private String[] cmdArray;

    private String[] envp;

    private File dir;

    public Command(String[] cmdArray) {
        this.cmdArray = cmdArray;
    }

    public Command(String[] cmdArray, String[] envp, File dir) {
        this.cmdArray = cmdArray;
        this.envp = envp;
        this.dir = dir;
    }

    ... 省略setters & getters
}
```

2. Win平台命令执行器

```java
/**
 * Author: Hunter Zhao
 * Date:   12/2/15 18:43
 * Description: command executor for Windows Platform
 */
public class WinExecutor extends CommandExecutor {

private static final String WIN_CMD = "C:\\Windows\\System32\\cmd.exe";

    public WinExecutor(String targetPath) {
        super.targetPath = targetPath;
    }

    @Override
    public String getTerminalPath() {
        return WIN_CMD;
    }

    @Override
    public Command buildCommand() {

        String terminalPath = this.getTerminalPath();
        String[] cmdArr = {terminalPath, "/k", "start", "cd", getTargetPath()};

        return new Command(cmdArr, null, new File(getTargetPath()));
    }
}
```

3. Mac平台命令执行器

```java
/**
 * Author: Hunter Zhao
 * Date:   12/2/15 18:42
 * Description: command executor for Mac Platform
 */
public class MacExecutor extends CommandExecutor {

    private static final String MAC_TERMINAL = "/Applications/Utilities/Terminal.app/Contents/MacOS/Terminal";
    private static final String ITERM        = "/Applications/iTerm.app/Contents/MacOS/iTerm";
    private static final String ITERM2       = "/Applications/iTerm.app/Contents/MacOS/iTerm2";

    public MacExecutor(String targetPath) {
        setTargetPath(targetPath);
    }

    @Override
    public String getTerminalPath() {

        if (isTerminalInstalled(ITERM)) {
            return ITERM;
        }
        else if (isTerminalInstalled(ITERM2)) {
            return ITERM2;
        }

        return MAC_TERMINAL;
    }

    @Override
    Command buildCommand() {
        String terminalPath = this.getTerminalPath();
        String[] cmdArr = {terminalPath, getTargetPath()};

        return new Command(cmdArr);
    }
}
```


## 调用系统命令行并切换至目标目录

```java
/**
 * Author: Hunter Zhao
 * Date:   12/2/15 18:41
 * Description: command execution template
 */
public abstract class CommandExecutor {

    /**
     * determine which terminal to be used
     * @return
     */
    abstract String getTerminalPath();

    /**
     * build the Command object
     * @return
     */
    abstract Command buildCommand();

    /** path to open */
    protected String targetPath;

    /**
     * determine if the specified terminal has been installed
     * @param terminalPath
     * @return
     */
    protected boolean isTerminalInstalled(String terminalPath) {
        File terminal = new File(terminalPath);
        return terminal.exists() && terminal.canExecute();
    }

    /**
     * open the target path in terminal
     */
    public void openTerminal() {

        Command cmd = this.buildCommand();

        try {
            Runtime.getRuntime().exec(cmd.getCmdArray(), cmd.getEnvp(), cmd.getDir());
        } catch (IOException e) {
            // ignored
        }
    };

    public String getTargetPath() {
        return targetPath;
    }

    public void setTargetPath(String targetPath) {
        this.targetPath = targetPath;
    }
}
```


## 开放插件入口

```java
/**
 * Author: Hunter Zhao
 * Date:   12/2/15 18:31
 * Description: An IntelliJ plugin for opening current directory in terminal
 */
public class OpenTerminalHereAction extends AnAction {

    public static final String PLUGIN_NAME = "OpenTerminalHere";

    @Override
    public void actionPerformed(AnActionEvent event) {

        Project project = event.getProject();
        if (project == null) {
            return;
        }

        perform(project);

    }

    /**
     * perform the action
     * @param project
     */
    private void perform(@NotNull Project project) {

        VirtualFile selectedFile = FileUtil.getSelectedProjectFile(project);
        if (selectedFile == null) {
            return;
        }

        String targetPath = selectedFile.getPath();
        CommandExecutor executor = null;
        if (SystemInfo.isMac) {
            executor = new MacExecutor(targetPath);
        }
        else if (SystemInfo.isWindows) {
            executor = new WinExecutor(targetPath);
        }

        if (executor == null) {
            NotificationTool.notify(project, PLUGIN_NAME,
                    "Your operating system is not supported temporarily.", NotificationType.ERROR);
            return;
        }

        executor.openTerminal();
    }
}
```

## 编辑plugin.xml配置

```xml
<actions>
    <!-- 插件基本信息 -->
    <action id="com.bobz.action.OpenTerminalHereAction" class="com.bobz.action.OpenTerminalHereAction"
            text="Open Terminal Here" description="Open Terminal Here">
        <!-- 添加至目标分组 -->
        <add-to-group group-id="ProjectViewPopupMenu" anchor="after" relative-to-action="RevealIn"/>
        <!-- 快捷键设置 -->
        <keyboard-shortcut keymap="$default" first-keystroke="ctrl alt T"/>
    </action>
</actions>
```


# 参考引用
- [IntelliJ Platform SDK](http://www.jetbrains.org/intellij/sdk/docs/welcome.html)

