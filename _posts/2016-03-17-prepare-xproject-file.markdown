---
layout: post
group: hook
title:  "Prepare ios xcodeproj file on Hooks listener"
date:   2016-03-17 11:32:36 +0100
categories: cordova hooks
---

# Push enabled by hooks

First we have to add our Hook file into our config.xml to activate it. A list of availables Hooks are [here](https://cordova.apache.org/docs/de/latest/guide/appdev/hooks/index.html){:target="_blank"}.

````xml
  <hook src="hooks/add_push_target_enabled.js" type="platform_add" />
````

Second we need the path to our ".xcodeproj" file, for this we need the Project name.

The information about the Project-Name can be found in your "config.xml" and looks like this.

````xml
<widget id="com.mway.pushtest" version="0.0.1" xmlns="http://www.w3.org/ns/widgets" xmlns:cdv="http://cordova.apache.org/ns/1.0">
    <name>Push Hook test</name>
````

I`am using "xml2js" and "fs for reading the xml.

xml2js is a npm package and can be installed with the following command

````shell
  npm i -D xml2js
````
Ok lets get the name :

````js
#!/usr/bin/env node --harmony
'use strict';
let fs = require('fs');
let xml2js = require('xml2js');
let join = require('path').join;

let rootdir = join(__dirname, '..');
let parser = new xml2js.Parser();

module.exports = function(context) {
  let platforms = context.opts.platforms;
  if (platforms.indexOf('ios') === -1) {
    return true;
  }

  fs.readFile(join(rootdir, 'config.xml'), function(err, data) {
    parser.parseString(data, function(err, result) {
      console.dir('Hi name', result.widget.name[0]);
    });
  });
};
````

Ok we have the Name of our Project dynamically. By the way checkout "context" there are a lot of informations about your project.
Next we have to get the rootPath of our ".xcodeproj" file and then we can use the npm package "xcode".

````js
  npm i -D xcode
````

i create a class for that :

```js
#!/usr/bin/env node --harmony
'use strict';
let fs = require('fs');
let xcode = require('xcode');
let join = require('path').join;
let xml2js = require('xml2js');

let rootdir = join(__dirname, '..');
let parser = new xml2js.Parser();

class Pbxproj {
  constructor(project) {
    this.project = `${project}.xcodeproj`;
    this.xcodeProject = null;
    this.rootPbxproj = this.pbxprojPath();

    if (!this.rootPbxproj) {
      console.error('project file not exist', this.rootPbxproj);
      process.exit()
    }
    console.log('Hey i have a .xcodeproj file found', this.rootPbxproj);
  }
  /**
   * root path to .xcodeproj file
   */
  pbxprojPath() {
    let temp = join(rootdir, 'platforms', 'ios', this.project, 'project.pbxproj');
    if (!fs.existsSync(temp)) {
      return null;
    }
    return temp;
  }
}

module.exports = function(context) {
  let platforms = context.opts.platforms;
  if (platforms.indexOf('ios') === -1) {
    return true;
  }
  fs.readFile(join(rootdir, 'config.xml'), function(err, data) {
    parser.parseString(data, function(err, result) {
      console.dir(result.widget.name[0]);
      new Pbxproj(result.widget.name[0]);
    });
  });
};
```
Ok now we have to parse the ".xcodeproj" file and set the credentials check the "parsePbxproj" method in the Full Code Example.

Full code example:

````js
#!/usr/bin/env node --harmony
'use strict';
let fs = require('fs');
let xcode = require('xcode');
let join = require('path').join;
let xml2js = require('xml2js');

let rootdir = join(__dirname, '..');
let parser = new xml2js.Parser();

class Pbxproj {
  constructor(project) {
    this.project = `${project}.xcodeproj`;
    this.xcodeProject = null;
    this.rootPbxproj = this.pbxprojPath();
    this.targetAttributes = {};
    if (!this.rootPbxproj) {
      console.error('project file not exist', this.rootPbxproj);
      process.exit()
    }
    this.parsePbxproj();
  }
  /**
   * root path to .xcodeproj file
   */
  pbxprojPath() {
    let temp = join(rootdir, 'platforms', 'ios', this.project, 'project.pbxproj');
    if (!fs.existsSync(temp)) {
      return null;
    }
    return temp;
  }
  /**
   * SystemCapabilities in there
   */
  hasSystemCapabilities(TargetAttributes) {
    if (TargetAttributes.SystemCapabilities) {
      return true;
    }
    return false;
  }
  /**
   * SystemCapabilities: {
   *  com.apple.Push
   */
  hasTargetAttributes(attributes, id) {
    if (!attributes) {
      return false;
    }
    if (attributes.TargetAttributes) {
      if (attributes.TargetAttributes[id]) {
        return true;
      }
    }
    return false;
  }
  /**
   * target uuid
   */
  getProjectTarget(pbxProject) {
    let target;
    let projectTargets = pbxProject.targets;
    for (let i = 0, j = pbxProject.targets.length; i < j; i++) {
      target = pbxProject.targets[i].value;
    }
    return target;
  }
  /**
   * filter objects where started with comment
   */
  getPbxProject(pbxProjectObj) {
    let pbxProject;
    for (let property in pbxProjectObj) {
      if (!/comment/.test(property)) {
        pbxProject = pbxProjectObj[property];
      }
    }
    return pbxProject;
  }
  /**
   * add Systemcapabilities
   */
  appendPush(pbxProjectObj) {
    let pbxProject = this.getPbxProject(pbxProjectObj);
    let target = this.getProjectTarget(pbxProject);
    if (!this.hasTargetAttributes(pbxProject.attributes, target)) {
      pbxProject.attributes.TargetAttributes = {};
      pbxProject.attributes.TargetAttributes[target] = { SystemCapabilities: {} };
    }
    // console.log('pbxProjectObj.attributes', pbxProject.attributes);
    if (this.hasSystemCapabilities(pbxProject.attributes.TargetAttributes[target])) {
      pbxProject.attributes.TargetAttributes[target].SystemCapabilities['com.apple.push'] = {
        enabled: 1
      };
    } else {
      pbxProject.attributes.TargetAttributes[target] = {
        SystemCapabilities: {
          "com.apple.push": {
            enabled: 1
          }
        }
      };
    }
    fs.writeFileSync(this.rootPbxproj, this.xcodeProject.writeSync());
  }
  /**
   * parse the pbxproj file
   */
  parsePbxproj() {
    this.xcodeProject = xcode.project(this.rootPbxproj);
    this.xcodeProject.addTargetDependency('')
    this.xcodeProject.parse(function(err) {
      if (err) {
        shell.echo('An error occured during parsing of [' + this.rootPbxproj + ']: ' + JSON.stringify(err, null, 2));
      } else {
        this.appendPush(this.xcodeProject.getPBXObject('PBXProject'));
      }
    }.bind(this));
  }
}

module.exports = function(context) {
  let platforms = context.opts.platforms;
  if (platforms.indexOf('ios') === -1) {
    return true;
  }
  fs.readFile(join(rootdir, 'config.xml'), function(err, data) {
    parser.parseString(data, function(err, result) {
      console.dir(result.widget.name[0]);
      new Pbxproj(result.widget.name[0]);
    });
  });
};
````
