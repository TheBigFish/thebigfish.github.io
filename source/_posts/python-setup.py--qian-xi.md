---
title: python setup.py 浅析
date: 2018-11-16 14:02:43
updated: 2018-11-28 13:36:36
tags: [python]
author: TheBigFish
---
# python setup.py 浅析

## setuptools.setup() 参数说明

### packages

对于所有 packages 列表里提到的纯 Python 模块做处理  
需要在 setup 脚本里有一个包名到目录的映射。  
默认对于 setup 脚本所在目录下同名的目录即视为包所在目录。  
当你在 setup 脚本中写入 packages = ['foo'] 时， setup 脚本的同级目录下可以找到 `foo/__init__.py`。如果没有找到对应文件，disutils 不会直接报错，而是给出一个告警然后继续进行有问题的打包流程。

### package_dir

阐明包名到目录的映射，见 packages

    package_dir = {'': 'lib'}

键: 代表了包的名字，空的包名则代表 root package(不在任何包中的顶层包)。  
值: 代表了对于 setup 脚本所在目录的相对路径.

    packages = ['foo']
    package_dir = {'': 'lib'}

指明包位于 lib/foo/, `lib/foo/__init__.py` 这个文件存在

另一种方法则是直接将 foo 这个包的内容全部放入 lib 而不是在 lib 下建一个 foo 目录

    package_dir = {'foo': 'lib'}

一个在 package_dir 字典中的 package: dir 映射会对当前包下的所有包都生效， 所以 foo.bar 会自动生效. 在这个例子当中， `packages = ['foo', 'foo.bar']` 告诉 distutils 去寻找 `lib/__init__.py` 和 `lib/bar/__init__.py`.

### py_modules

对于一个相对较小的模块的发布，你可能更想要列出所有模块而不是列出所有的包，尤其是对于那种根目录下就是一个简单模块的类型.
这描述了两个包，一个在根目录下，另一个则在 pkg 目录下。
默认的 “包：目录” 映射关系表明你可以在 setup 脚本所在的路径下找到 mod1.py 和 pkg/mod2.py。  
当然，你也可以用 package_dir 选项重写这层映射关系就是了。

### find_packages

packages=find_packages(exclude=('tests', 'robot_server.scripts')),
exclude 里面是包名，而非路径

### include_package_data

引入包内的非 Python 文件  
include_package_data 需要配合 MANIFEST.in 一起使用

MANIFEST.in:

```python
include myapp/scripts/start.py
recursive-include myapp/static *
```

```python
setup(
    name='MyApp',         # 应用名
    version='1.0',        # 版本号
    packages=['myapp'],   # 包括在安装包内的Python包
    include_package_data=True    # 启用清单文件MANIFEST.in
)
```

注意，此处引入或者排除的文件必须是 package 内的文件

    setup-demo/
      ├ mydata.data      # 数据文件
      ├ setup.py         # 安装文件
      ├ MANIFEST.in      # 清单文件
      └ myapp/           # 源代码
          ├ static/      # 静态文件目录
          ├ __init__.py
          ...

在 MANIFEST.in 引入 include mydata.data 将不起作用

### exclude_package_date

排除一部分包文件
{'myapp':['.gitignore]}，就表明只排除 myapp 包下的所有. gitignore 文件。

### data_files

指定其他的一些文件（如配置文件）

```python
data_files=[('bitmaps', ['bm/b1.gif', 'bm/b2.gif']),
            ('config', ['cfg/data.cfg']),
            ('/etc/init.d', ['init-script'])]
```

规定了哪些文件被安装到哪些目录中。  
如果目录名是相对路径 (比如 bitmaps)，则是相对于 sys.prefix(/usr) 或 sys.exec_prefix 的路径。  
否则安装到绝对路径 (比如 /etc/init.d)。

### cmdclass

定制化命令，通过继承 setuptools.command 下的命令类来进行定制化

```python
class UploadCommand(Command):
    """Support setup.py upload."""
    ...

    def run(self):
        try:
            self.status('Removing previous builds…')
            rmtree(os.path.join(here, 'dist'))
        except OSError:
            pass

        self.status('Building Source and Wheel (universal) distribution…')
        os.system('{0} setup.py sdist bdist_wheel --universal'.format(sys.executable))

        self.status('Uploading the package to PyPI via Twine…')
        os.system('twine upload dist/*')

        self.status('Pushing git tags…')
        os.system('git tag v{0}'.format(about['__version__']))
        os.system('git push --tags')

        sys.exit()

setup(
    ...
    # $ setup.py publish support.
    cmdclass={
        'upload': UploadCommand,
    },
)
```

这样可以通过 `python setup.py upload` 运行打包上传代码

### install_requires

安装这个包所需要的依赖，列表

### tests_require

与 install_requires 作用相似，单元测试时所需要的依赖

## 虚拟运行环境下安装包

以 [legit](https://github.com/kennethreitz/legit) 为例

-   下载 lgit 源码  
    `git clone https://github.com/kennethreitz/legit.git`

-   创建虚拟运行环境  
    `virtualenv --no-site-packages venv`  
    运行环境目录结构为：

        venv/
        ├── bin
        ├── include
        ├── lib
        ├── local
        └── pip-selfcheck.json

-   打包工程  
    `python3 setup.py sdist bdist_wheel`

        .
        ├── AUTHORS
        ├── build
        │   ├── bdist.linux-x86_64
        │   └── lib.linux-x86_64-2.7
        ├── dist
        │   ├── legit-1.0.1-py2.py3-none-any.whl
        │   └── legit-1.0.1.tar.gz

    在 dist 下生成了安装包

-   进入虚拟环境  
    `source venv/bin/activate`

-   安装包  
     `pip install ./dist/legit-1.0.1.tar.gz`

        Successfully built legit args clint
        Installing collected packages: appdirs, args, click, lint, colorama, crayons, smmap2, gitdb2, GitPython, ix, pyparsing, packaging, legit
        Successfully installed GitPython-2.1.8 appdirs-1.4.3 rgs-0.1.0 click-6.7 clint-0.5.1 colorama-0.4.0 rayons-0.1.2 gitdb2-2.0.3 legit-1.0.1 packaging-17.1 yparsing-2.2.0 six-1.11.0 smmap2-2.0.3

## 安装过程分析

`venv/lib/python2.7/site-packages/` 下安装了 legit 及依赖包

    legit/venv/lib/python2.7/site-packages$ tree -L 1

    .
    ├── appdirs-1.4.3.dist-info
    ├── appdirs.py
    ├── appdirs.pyc
    ├── args-0.1.0.dist-info
    ├── args.py
    ├── args.pyc
    ├── click
    ├── click-6.7.dist-info
    ├── clint
    ├── clint-0.5.1.dist-info
    ├── colorama
    ├── colorama-0.4.0.dist-info
    ├── crayons-0.1.2.dist-info
    ├── crayons.py
    ├── crayons.pyc
    ├── easy_install.py
    ├── easy_install.pyc
    ├── git
    ├── gitdb
    ├── gitdb2-2.0.3.dist-info
    ├── GitPython-2.1.8.dist-info
    ├── legit
    ├── legit-1.0.1.dist-info
    ├── packaging
    ├── packaging-17.1.dist-info
    ├── pip
    ├── pip-18.1.dist-info
    ├── pkg_resources
    ├── pyparsing-2.2.0.dist-info
    ├── pyparsing.py
    ├── pyparsing.pyc
    ├── setuptools
    ├── setuptools-40.6.2.dist-info
    ├── six-1.11.0.dist-info
    ├── six.py
    ├── six.pyc
    ├── smmap
    ├── smmap2-2.0.3.dist-info
    ├── wheel
    └── wheel-0.32.2.dist-info

`venv/bin` 下新增可执行文件 legit, 内容为

```python
#!/home/turtlebot/learn/python/legit/venv/bin/python

# -*- coding: utf-8 -*-
import re
import sys

from legit.cli import cli

if __name__ == '__main__':
    sys.argv[0] = re.sub(r'(-script\.pyw?|\.exe)?$', '', sys.argv[0])
    sys.exit(cli())
```

此时，可以直接运行

```bash
>>> legit
```

## setup.py 分析

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import os
import sys
from codecs import open  # To use a consistent encoding

from setuptools import setup  # Always prefer setuptools over distutils

APP_NAME = 'legit'
APP_SCRIPT = './legit_r'
VERSION = '1.0.1'


# Grab requirements.
with open('reqs.txt') as f:
    required = f.readlines()


settings = dict()


# Publish Helper.
if sys.argv[-1] == 'publish':
    os.system('python setup.py sdist bdist_wheel upload')
    sys.exit()


if sys.argv[-1] == 'build_manpage':
    os.system('rst2man.py README.rst > extra/man/legit.1')
    sys.exit()


# Build Helper.
if sys.argv[-1] == 'build':
    import py2exe  # noqa
    sys.argv.append('py2exe')

    settings.update(
        console=[{'script': APP_SCRIPT}],
        zipfile=None,
        options={
            'py2exe': {
                'compressed': 1,
                'optimize': 0,
                'bundle_files': 1}})

settings.update(
    name=APP_NAME,
    version=VERSION,
    description='Git for Humans.',
    long_description=open('README.rst').read(),
    author='Kenneth Reitz',
    author_email='me@kennethreitz.com',
    url='https://github.com/kennethreitz/legit',
    packages=['legit'],
    install_requires=required,
    license='BSD',
    classifiers=[
        'Development Status :: 5 - Production/Stable',
        'Intended Audience :: Developers',
        'Natural Language :: English',
        'License :: OSI Approved :: BSD License',
        'Programming Language :: Python',
        'Programming Language :: Python :: 2',
        'Programming Language :: Python :: 2.7',
        'Programming Language :: Python :: 3',
        'Programming Language :: Python :: 3.4',
        'Programming Language :: Python :: 3.5',
        'Programming Language :: Python :: 3.6',
    ],
    entry_points={
        'console_scripts': [
            'legit = legit.cli:cli',
        ],
    }
)


setup(**settings)
```

-   packages=['legit'] 引入 legit 目录下的所有默认引入文件
-   install_requires=required 指明安装时需要额外安装的第三方库
-   `'console_scripts': ['legit = legit.cli:cli',]` 生成可执行控制台程序，程序名为 legit, 运行 legit.cli 中的 cli() 函数。最终会在 bin/ 下生成 legit 可执行 py 文件，调用制定的函数

## setup.py 实例分析

[kennethreitz/setup.py](https://github.com/kennethreitz/setup.py)

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

# Note: To use the 'upload' functionality of this file, you must:
#   $ pip install twine

import io
import os
import sys
from shutil import rmtree

from setuptools import find_packages, setup, Command

# Package meta-data.
NAME = 'mypackage'
DESCRIPTION = 'My short description for my project.'
URL = 'https://github.com/me/myproject'
EMAIL = 'me@example.com'
AUTHOR = 'Awesome Soul'
REQUIRES_PYTHON = '>=3.6.0'
VERSION = None

# What packages are required for this module to be executed?
REQUIRED = [
    # 'requests', 'maya', 'records',
]

# What packages are optional?
EXTRAS = {
    # 'fancy feature': ['django'],
}

# The rest you shouldn't have to touch too much :)
# ------------------------------------------------
# Except, perhaps the License and Trove Classifiers!
# If you do change the License, remember to change the Trove Classifier for that!

here = os.path.abspath(os.path.dirname(__file__))

# Import the README and use it as the long-description.
# Note: this will only work if 'README.md' is present in your MANIFEST.in file!
try:
    with io.open(os.path.join(here, 'README.md'), encoding='utf-8') as f:
        long_description = '\n' + f.read()
except FileNotFoundError:
    long_description = DESCRIPTION

# Load the package's __version__.py module as a dictionary.
about = {}
if not VERSION:
    with open(os.path.join(here, NAME, '__version__.py')) as f:
        exec(f.read(), about)
else:
    about['__version__'] = VERSION


class UploadCommand(Command):
    """Support setup.py upload."""

    description = 'Build and publish the package.'
    user_options = []

    @staticmethod
    def status(s):
        """Prints things in bold."""
        print('\033[1m{0}\033[0m'.format(s))

    def initialize_options(self):
        pass

    def finalize_options(self):
        pass

    def run(self):
        try:
            self.status('Removing previous builds…')
            rmtree(os.path.join(here, 'dist'))
        except OSError:
            pass

        self.status('Building Source and Wheel (universal) distribution…')
        os.system('{0} setup.py sdist bdist_wheel --universal'.format(sys.executable))

        self.status('Uploading the package to PyPI via Twine…')
        os.system('twine upload dist/*')

        self.status('Pushing git tags…')
        os.system('git tag v{0}'.format(about['__version__']))
        os.system('git push --tags')

        sys.exit()


# Where the magic happens:
setup(
    name=NAME,
    version=about['__version__'],
    description=DESCRIPTION,
    long_description=long_description,
    long_description_content_type='text/markdown',
    author=AUTHOR,
    author_email=EMAIL,
    python_requires=REQUIRES_PYTHON,
    url=URL,
    packages=find_packages(exclude=('tests',)),
    # If your package is a single module, use this instead of 'packages':
    # py_modules=['mypackage'],

    # entry_points={
    #     'console_scripts': ['mycli=mymodule:cli'],
    # },
    install_requires=REQUIRED,
    extras_require=EXTRAS,
    include_package_data=True,
    license='MIT',
    classifiers=[
        # Trove classifiers
        # Full list: https://pypi.python.org/pypi?%3Aaction=list_classifiers
        'License :: OSI Approved :: MIT License',
        'Programming Language :: Python',
        'Programming Language :: Python :: 3',
        'Programming Language :: Python :: 3.6',
        'Programming Language :: Python :: Implementation :: CPython',
        'Programming Language :: Python :: Implementation :: PyPy'
    ],
    # $ setup.py publish support.
    cmdclass={
        'upload': UploadCommand,
    },
)
```


***
Sync From: https://github.com/TheBigFish/blog/issues/6
