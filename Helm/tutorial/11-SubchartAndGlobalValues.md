# Subcharts and Global Values
To this point we have been working only with one chart. But ***charts can have dependencies***, called ***subcharts***, that also have their own values and templates.  
In this section we will create a subchart and see the different ways we can access values from within templates.

Before we dive into the code, there are a few important details to learn about application subcharts.

1.  subchart is considered **"stand-alone"**, which means a subchart can never explicitly depend on its parent chart.
2. For that reason, a subchart cannot access the values of its parent.
3. A parent chart can override values for subcharts.
4. Helm has a concept of global values that can be accessed by all charts.

These limitations do not all necessarily apply to library charts, which are designed to provide standardized helper functionality.
As we walk through the examples in this section, many of these concepts will become clearer.

## Creating a Subchart
For these exercises, we'll start with the mychart/ chart we created at the beginning of this guide, and we'll add a new chart inside of it.

```
$ cd mychart/charts
$ helm create mysubchart
Creating mysubchart
$ rm -rf mysubchart/templates/*
```

Notice that just as before, we deleted all of the base templates so that we can start from scratch. In this guide, we are focused on how templates work, not on managing dependencies. But the Charts Guide has more information on how subcharts work.

## Adding Values and a Template to the Subchart
Next, let's create a simple template and values file for our mysubchart chart. There should already be a *values.yaml* in *mychart/charts/mysubchart*. We'll set it up like this:

```
dessert: cake
```

Next, we'll create a new ConfigMap template in mychart/charts/mysubchart/templates/configmap.yaml:

```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-cfgmap2
data:
  dessert: {{ .Values.dessert }}
```

Because every subchart is a stand-alone chart, we can test mysubchart on its own:

```
$ helm install --generate-name --dry-run mychart/charts/mysubchart
SERVER: "localhost:44134"
CHART PATH: /Users/mattbutcher/Code/Go/src/helm.sh/helm/_scratch/mychart/charts/mysubchart
NAME:   newbie-elk
TARGET NAMESPACE:   default
CHART:  mysubchart 0.1.0
MANIFEST:
---
# Source: mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: newbie-elk-cfgmap2
data:
  dessert: cake
```
# Overriding Values from a Parent Chart
Our original chart, mychart is now the parent chart of mysubchart. This relationship is based entirely on the fact that mysubchart is within *mychart/charts*.

Because *mychart* is a parent, we can specify configuration in mychart and have that configuration pushed into mysubchart. For example, we can modify *mychart/values.yaml* like this:

```
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions

mysubchart:
  dessert: ice cream
```

Note the last two lines. Any directives inside of the *mysubchart* section will be sent to the mysubchart chart.  
So if we run :

```
$ helm install --dry-run -generate-name mychart
``` 

one of the things we will see is the mysubchart ConfigMap:
```

NAME: mychart-1693321837
LAST DEPLOYED: Tue Aug 29 17:10:37 2023
NAMESPACE: helmtest
STATUS: pending-install
REVISION: 1
TEST SUITE: None
HOOKS:
MANIFEST:
---
# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-1693321837-cfgmap2
data:
  dessert: ice cream
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-1693321837-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  mug: "true"
```

The value at the top level has now overridden the value of the subchart.

There's an important detail to notice here. **We didn't change the template of mychart/charts/mysubchart/templates/configmap.yaml to point to .Values.mysubchart.dessert.** From that template's perspective, the value is still located at .Values.dessert. As the template engine passes values along, it sets the scope. So for the mysubchart templates, only values specifically for mysubchart will be available in .Values.

*Sometimes, though, you do want certain values to be available to all of the templates. This is accomplished using **global** chart values.*

# Global Chart Values
Global values are values that can be accessed from any chart or subchart by exactly the same name. Globals require explicit declaration. You can't use an existing non-global as if it were a global.

The Values data type has a reserved section called **Values.global** where global values can be set.  
Let's set one in our mychart/values.yaml file.
```
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions

mysubchart:
  dessert: ice cream

global:
  salad: caesar
```

Because of the way globals work, both mychart/templates/configmap.yaml and mysubchart/templates/configmap.yaml should be able to access that value as ```{{ .Values.global.salad }}```.

***mychart/templates/configmap.yaml***

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  salad: {{ .Values.global.salad }}
```

***mysubchart/templates/configmap.yaml***

```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-cfgmap2
data:
  dessert: {{ .Values.dessert }}
  salad: {{ .Values.global.salad }}
```

Now if we run a dry run install, we'll see the same value in both outputs:

```
NAME: mychart-1693322514
LAST DEPLOYED: Tue Aug 29 17:21:54 2023
NAMESPACE: helmtest
STATUS: pending-install
REVISION: 1
TEST SUITE: None
HOOKS:
MANIFEST:
---
# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-1693322514-cfgmap2
data:
  dessert: ice cream
  salad: caesar
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-1693322514-configmap
data:
  salad: caesar

```

Globals are useful for passing information like this, though it does take some planning to make sure the right templates are configured to use globals.

## Sharing Templates with Subcharts
Parent charts and subcharts can share templates. Any defined block in any chart is available to other charts.

For example, we can define a simple template like this:

```
{{- define "parentchart-labels" -}}
app: {{ .Chart.Name }}
{{- end }}
```

The above-defined template can be used by any of the sub-charts. A named template can be embedded using includeor template keyword.

***mychart/templates/configmap.yaml***

```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
  {{ include "parentchart.labels" . | indent 4 }}
data:
  salad: {{ .Values.global.salad }}
```

***mychart/charts/mysubchart/templates/configmap.yaml***

```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-cfgmap2
  labels:
  {{- include "parentchart.labels" . | nindent 4 }}
data:
  dessert: {{ .Values.dessert }}
  salad: {{ .Values.global.salad }}

```

When we render this we get the following output

```
$ helm install --generate-name --dry-run mychart
NAME: mychart-1693324683
LAST DEPLOYED: Tue Aug 29 17:58:03 2023
NAMESPACE: helmtest
STATUS: pending-install
REVISION: 1
TEST SUITE: None
HOOKS:
MANIFEST:
---
# Source: mychart/charts/mysubchart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-1693324683-cfgmap2
  labels:
    app: mysubchart
data:
  dessert: ice cream
  salad: caesar
---
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mychart-1693324683-configmap
  labels:
      app: mychart
data:
  salad: caesar
```

Note that both configmaps have access to the define template. Each of them replace the value with the respective chart name.


While chart developers have a choice between include and template, one advantage of using include is that include can dynamically reference templates:

```{{ include $mytemplate }}```

The above will dereference ***$mytemplate***. The template function, in contrast, will only accept a string literal.

## Avoid Using Blocks
The Go template language provides a block keyword that allows developers to provide a default implementation which is overridden later. In Helm charts, blocks are not the best tool for overriding because if multiple implementations of the same block are provided, the one selected is unpredictable.

The suggestion is to instead use ***include***.

