# Uniform #
### 1、Start ###
**基本用法**
##
	// Choose your theme
	import AutoForm from 'uniforms-unstyled/AutoForm';
	
	// A compatible schema
	import PostSchema from './schemas/Post';
	
	// This will render an automatic, validated form, with labelled fields, inline
	// validation and a submit button. If model will be present, form will be filled
	// with appropriate values.
	const PostForm = ({model}) =>
	<AutoForm schema={PostSchema} onSubmit={doc => db.save(doc)} model={model} />
	;
> 自动验证和操纵model的自动改变

**Form中自定义layout**
##
	// Choose your theme
	import AutoField   from 'uniforms-unstyled/AutoField';
	import AutoForm    from 'uniforms-unstyled/AutoForm';
	import SubmitField from 'uniforms-unstyled/SubmitField';
	import TextField   from 'uniforms-unstyled/TextField';
	
	// A compatible schema
	import PostSchema from './schemas/Post';
	
	const PostForm = ({model}) =>
	    <AutoForm schema={PostSchema} onSubmit={doc => db.save(doc)} model={model}>
	        <h2>Title</h2>
	
	        <AutoField name="fieldA" />
	        <TextField name="fieldB" />
	
	        <div className="super-special-class">
	            <SubmitField className="super-special-class-with-suffix" />
	        </div>
	    </AutoForm>
	;


### 2、Overview ###
**Forms components**
##
- 常用的是 AutoForm 和 ValidatedForm

![](https://i.imgur.com/hhFZ8TA.jpg)

- 层级关系（Hierarchy）

![](https://i.imgur.com/s6wrvjI.png)


**Fields components**
##
![](https://i.imgur.com/vPL5CHj.png)

### 3、Advanced topics ###
**Forms**
##
- 异步验证（Asynchronous validation）

ValidatedForm和AutoForm都有一个onValidate的值，能创建如下的验证器

		const onValidate = (model, error, callback) => {
		    // You can either ignore validation error...
		    if (omitValidation(model)) {
		        return callback(null);
		    }
		
		    // ...or any additional validation if an error is already there...
		    if (error) {
		        return callback();
		    }
		
		    // ...or feed it with another error.
		    MyAPI.validate(model, error => callback(error || null));
		};
		
		// Later...
		
		<ValidatedForm {...props} onValidate={onValidate} />


- 自动保存（Autosave）

每个表单都有一个自动保存的函数，每次改变值都会触发submit。延迟自动保存，可设置延迟时间autosaveDelay，默认为0。

	<AutoForm
	    autosave
	    autosaveDelay={5000} // 5 seconds
	    schema={schema}
	    onSubmit={onSubmit}
	/>
- 方法

	1. change(key, value)
	1. reset()
	1. submit()
	1. validate() (added in ValidatedForm)
	
可以使用react的ref去访问到form中的方法，如下：

	const MyForm = ({schema, onSubmit}) => {
    let formRef;

    return (
        <section>
            <AutoForm ref={ref => formRef = ref} schema={schema} onSubmit={onSubmit} />
            <small onClick={() => formRef.reset()}>
                Reset
            </small>
            <small onClick={() => formRef.submit()}>
                Submit
            </small>
        </section>
    );
	};

- 反应关联

如果你想要改变一个field，影响另外一个field的值，只要继承AutoForm,重写onChange
	
	class ChainForm extends AutoForm {
	    onChange (key, value) {
	        if (key === 'key_to_intercept') return;
	        if (key === 'key_to_translate') return super.onChange('another_key', value);
	        if (key === 'key_to_mutate') {
	            super.onChange('another_key1', value * 2);
	            super.onChange('another_key2', value / 2);
	            return;
	        }
	
	        super.onChange(key, value);
	    }
	}

它可以很容易将表单重用

	const withMultipliedField = (fieldA, fieldB, Form) =>
	class withMultipliedFieldForm extends Form {
	    onChange (key, value) {
	        // Multiply fieldA
	        if (key === fieldA)
	            super.onChange(fieldB, value + value);
	
	        // Pass every change
	        super.onChange(key, value);
	    }
	};


- model转变（Model transformations）

如果你想要在表单被验证或被提交，再或者传递到子域之前改变model，可以使用
modelTransform

	<AutoForm
	    // Do not mutate given model!
	    modelTransform={(mode, model) => {
	        // This model will be passed to the fields.
	        if (mode === 'form') {/* ... */}
	
	        // This model will be submitted.
	        if (mode === 'submit') {/* ... */}
	
	        // This model will be validated.
	        if (mode === 'validate') {/* ... */}
	
	        // Otherwise, return unaltered model.
	        return model;
	    }}
	    onSubmit={onSubmit}
	    schema={schema}
	/>

- Post提交

成功回掉函数onSubmitSuccess，失败回掉函数onSubmitFailure
	
	<AutoForm
	    schema={schema}
	    onSubmit={doc => db.saveThatReturnsPromise(doc)}
	    onSubmitSuccess={() => alert('Promise resolved!')}
	    onSubmitFailure={() => alert('Promise rejected!')}
	/>

- 验证的options和方式

① modes

1. onChange （Validate on every change.每次改变验证）
1. onChangeAfterSubmit (default) Validate on every change, but only after first submit.（第一次提交验证）
1. onSubmit （Validate on every submit.每次提交验证）

② 若验证器有多个选项，可使用validator

	<AutoForm
	    validate="onChange"
	    validator={validatorOptions}
	    schema={schema}
	    onSubmit={onSubmit}
	/>

- ModifierForm  重写form方法的例子

		import BaseForm from 'uniforms/BaseForm';
		
		// In uniforms, every form is just an injectable set of functionalities. Thus,
		// we can live without many higher order components, using composed ones
		// instead. If you want to get a deeper dive into it, read the source of
		// AutoForm or QuickForm in the core package.
		const Modifier = parent => class extends parent {
	    // Expose injector.
	    //   It's not required, but recommended.
	    static Modifier = Modifier;
	
	    // Alter component display name.
	    //   It's not required, but recommended.
	    static displayName = `Modifier${parent.displayName}`;
	
	    // Here you can override any form methods or create additional ones.
	    getModel (mode) {
	        if (mode === 'submit') {
	            const doc  = super.getModel('submit');
	            const keys = this.getChildContextSchema().getSubfields();
	
	            const update = keys.filter(key => doc[key] !== undefined);
	            const remove = keys.filter(key => doc[key] === undefined);
	
	            // It's a good idea to omit empty modifiers.
	            const $set   = update.reduce((acc, key) => ({...acc, [key]: doc[key]}), {});
	            const $unset = remove.reduce((acc, key) => ({...acc, [key]: ''}), {});
	
	            return {$set, $unset};
	        }
	
	        return super.getModel(mode);
	    }
		};
		// Now we have to inject our functionality. This one is a ModifierForm. Use any
		// form component you want.
		export default Modifier(BaseForm);

**Fields**
##
- AutoField的算法（AutoField algorithm）
	
		let component = props.component;
		if (component === undefined) {
		    if (props.allowedValues) {
		        if (props.checkboxes && props.fieldType !== Array) {
		            component = RadioField;
		        } else {
		            component = SelectField;
		        }
		    } else {
		        switch (props.fieldType) {
		            case Date:    component = DateField; break;
		            case Array:   component = ListField; break;
		            case Number:  component = NumField;  break;
		            case Object:  component = NestField; break;
		            case String:  component = TextField; break;
		            case Boolean: component = BoolField; break;
		        }
		
		        invariant(component, 'Unsupported field type: %s', props.fieldType.toString());
		    }
		}

- props

![](https://i.imgur.com/wFftVcM.png)

	<TextField />                    // default label | no      placeholder
	<TextField label="Text" />       // custom  label | no      placeholder
	<TextField label={false} />      // no      label | no      placeholder
	<TextField placeholder />        // default label | default placeholder
	<TextField placeholder="Text" /> // default label | custom  placeholder
	
	<NestField label={null}> // null = no label but the children have their labels
	    <TextField />
	</NestField>
	
	<NestField label={false}> // false = no label and the children have no labels
	    <TextField />
	</NestField>
	
	<ListField name="authors" disabled>   // Additions are disabled...
	    <ListItemField name="$" disabled> // ...deletion too
	        <NestField disabled={false}>  // ...but editing is not.
	            <TextField name="name" />
	            <NumField  name="age" />
	        </NestField>
	    </ListItemField>
	</ListField>

- CompositeField  
详情请见
[https://github.com/vazco/uniforms/blob/master/API.md#connectfield](https://github.com/vazco/uniforms/blob/master/API.md#connectfield)

		import AutoField    from 'uniforms/AutoField';
		import React        from 'react';
		import connectField from 'uniforms/connectField';
		
		// This field is a kind of a shortcut for few fields. You can also access all
		// field props here, like value or onChange for some extra logic.
		const Composite = () =>
		    <section>
		        <AutoField field="firstName" />
		        <AutoField field="lastName" />
		        <AutoField field="age" />
		    </section>
		;
		
		export default connectField(Composite);

- CustomAutoField
> These are two standard options to define a custom AutoField: either using connectField or simply taking the code from the original one (theme doesn't matter) and simply apply own components and/or rules to render components. Below an example with connectField.
> 
> Note: This example uses connectField helper. To read more see API.
		// Remember to choose a correct theme package
		import AutoField from 'uniforms-unstyled/AutoField';
		
		const CustomAuto = props => {
		    // This way we don't care about unhandled cases - we use default
		    // AutoField as a fallback component.
		    const Component = determineComponentFromProps(props) || AutoField;
		
		    return (
		        <Component {...props} />
		    );
		};
		
		const CustomAutoField = connectField(CustomAuto, {
		    ensureValue:    false,
		    includeInChain: false,
		    initialValue:   false
		});

你也可以使用它在AutoForm / QuickForm / ValidatedQuickForm中

	<AutoForm {...props} autoField={CustomAutoField} />

- CycleField

> This example uses connectField helper.


		import React        from 'react';
		import classnames   from 'classnames';
		import connectField from 'uniforms/connectField';
		
		// This field works as follows: iterate all allowed values and optionally no-value
		// state if the field is not required. This one uses Semantic-UI.
		const Cycle = ({allowedValues, disabled, label, required, value, onChange}) =>
		    <a
		        className={classnames('ui', !value && 'basic', 'label')}
		        onClick={() =>
		            onChange(value
		                ? allowedValues.indexOf(value) === allowedValues.length - 1
		                    ? required
		                        ? allowedValues[0]
		                        : null
		                    : allowedValues[allowedValues.indexOf(value) + 1]
		                : allowedValues[0]
		            )
		        }
		    >
		        {value || label}
		    </a>
		;

		export default connectField(Cycle);

- RangeField

> This example uses connectField helper.

一个开始时间与结束时间选择器

	import React        from 'react';
	import connectField from 'uniforms/connectField';
	
	// This field works as follows: two datepickers are bound to each other. Value is
	// a {start, stop} object.
	const Range = ({onChange, value: {start, stop}}) =>
	    <section>
	        <DatePicker max={stop}  value={start} onChange={start => onChange(start, stop)} />
	        <DatePicker min={start} value={stop}  onChange={stop  => onChange(start, stop)} />
	    </section>
	;

	export default connectField(Range);

- RatingField
> This example uses connectField helper.

	import React        from 'react';
	import classnames   from 'classnames';
	import connectField from 'uniforms/connectField';
	
	// This field works as follows: render stars for each rating and mark them as
	// filled, if rating (value) is greater. This one uses Semantic-UI.
	const Rating = ({className, disabled, max = 5, required, value, onChange}) =>
	    <section className={classnames('ui', {disabled, required}, className, 'rating')}>
	        {[...Array(max)].map((_, index) => index + 1).map(index =>
	            <i
	                key={index}
	                className={classnames(index <= value && 'active', 'icon')}
	                onClick={() => disabled || onChange(!required && value === index ? null : index)}
	            />
	        )}
	    </section>
	;
	
	export default connectField(Rating);

**Schemas**
##
- method

	1. getError(name, error)
	1. getErrorMessage(name, error)
	1. getErrorMessages(error)
	1. getField(name)
	1. getInitialValue(name, props)
	1. getProps(name, props)
	1. getSubfields(name)
	1. getType(name)
	1. getValidator(options)

详情请看
[https://github.com/vazco/uniforms/blob/master/API.md#bridge](https://github.com/vazco/uniforms/blob/master/API.md#bridge)

- Currently built-in bridges:
	1. GraphQLBridge
	1. SimpleSchemaBridge
	1. SimpleSchema2Bridge
	
① GraphQLBridge
	
	import GraphQLBridge    from 'uniforms/GraphQLBridge';
	import {buildASTSchema} from 'graphql';
	import {parse}          from 'graphql';
	
	const schema = `
	    type Author {
	        id:        String!
	        firstName: String
	        lastName:  String
	    }
	
	    type Post {
	        id:     Int!
	        author: Author!
	        title:  String
	        votes:  Int
	    }
	
	    # This is required by buildASTSchema
	    type Query { anything: ID }
	`;
	
	const schemaType = buildASTSchema(parse(schema)).getType('Post');
	const schemaData = {
	    id: {
	        allowedValues: [1, 2, 3]
	    },
	    title: {
	        options: [
	            {label: 1, value: 'a'},
	            {label: 2, value: 'b'}
	        ]
	    }
	};
	
	const schemaValidator = model => {
	    const details = [];
	
	    if (!model.id) {
	        details.push({name: 'id', message: 'ID is required!'});
	    }
	
	    // ...
	
	    if (details.length) {
	        throw {details};
	    }
	};
	
	const bridge = new GraphQLBridge(schemaType, schemaValidator, schemaData);
	
	// Later...
	
	<ValidatedForm schema={bridge} />

②SimpleSchema definition

首先要导入uniforms的包

	const PersonSchema = new SimpleSchema({
	    // ...
	
	    aboutMe: {
	        type: String,
	        uniforms: MyText       // Component...
	        uniforms: {            // ...or object...
	            component: MyText, // ...with component...
	            propA: 1           // ...and/or extra props.
	        }
	    }
	});

③创建自己的schema bridges
	import Bridge from 'uniforms/Bridge';
	
	class MyLittleSchema extends Bridge {
	    constructor (schema, validator) {
	        super();
	
	        this.schema    = schema;
	        this.validator = validator;
	    }
	
	    getError (name, error) {
	        return error && error[name];
	    }
	
	    getErrorMessage (name, error) {
	        return error && error[name];
	    }
	
	    getErrorMessages (error) {
	        return error
	            ? Object.keys(this.schema).map(field => error[field])
	            : [];
	    }
	
	    getField (name) {
	        return this.schema[name.replace(/\.\d+/g, '.$')];
	    }
	
	    getType (name) {
	        return this.schema[name.replace(/\.\d+/g, '.$')].__type__;
	    }
	
	    getProps (name) {
	        return this.schema[name.replace(/\.\d+/g, '.$')];
	    }
	
	    getInitialValue (name) {
	        return this.schema[name.replace(/\.\d+/g, '.$')].initialValue;
	    }
	
	    getSubfields (name) {
	        return name
	            ? this.schema[name.replace(/\.\d+/g, '.$')].subfields || []
	            : Object.keys(this.schema).filter(field => field.indexOf('.') === -1);
	    }
	
	    getValidator () {
	        return this.validator;
	    }
	}
	
	const bridge = new MyLittleSchema({
	    login:     {__type__: String, required: true, initialValue: '', label: 'Login'},
	    password1: {__type__: String, required: true, initialValue: '', label: 'Password'},
	    password2: {__type__: String, required: true, initialValue: '', label: 'Password (again)'}
	}, model => {
	    const error = {};
	
	    if (!model.login) {
	        error.login = 'Login is required!';
	    } else if (model.login.length < 5) {
	        error.login = 'Login has to be at least 5 characters long!';
	    }
	
	    if (!model.password1) {
	        error.password1 = 'Password is required!';
	    } else if (model.password1.length < 10) {
	        error.login = 'Password has to be at least 10 characters long!';
	    }
	
	    if (model.password1 !== model.password2) {
	        error.password1 = 'Passwords mismatch!';
	    }
	
	    if (Object.keys(error).length) {
	        throw error;
	    }
	});
	
	<AutoForm schema={bridge} />

**Context data**
##
- Available context data 容器的表单内容类型

		MyComponentUsingUniformsContext.contextTypes = {
	    uniforms: PropTypes.shape({
	        name: PropTypes.arrayOf(PropTypes.string).isRequired,
	        error: PropTypes.any,
	        model: PropTypes.object.isRequired,
	
	        schema: PropTypes.shape({
	            getError:         PropTypes.func.isRequired,
	            getErrorMessage:  PropTypes.func.isRequired,
	            getErrorMessages: PropTypes.func.isRequired,
	            getField:         PropTypes.func.isRequired,
	            getInitialValue:  PropTypes.func.isRequired,
	            getProps:         PropTypes.func.isRequired,
	            getSubfields:     PropTypes.func.isRequired,
	            getType:          PropTypes.func.isRequired,
	            getValidator:     PropTypes.func.isRequired
	        }).isRequired,
	
	        state: PropTypes.shape({
	            changed:    PropTypes.bool.isRequired,
	            changedMap: PropTypes.object.isRequired,
	
	            label:       PropTypes.bool.isRequired,
	            disabled:    PropTypes.bool.isRequired,
	            placeholder: PropTypes.bool.isRequired
	        }).isRequired,
	
	        onChange: PropTypes.func.isRequired,
	        randomId: PropTypes.func.isRequired
	    }).isRequired
		};

-  DisplayIf（Example1）

		import BaseField  from 'uniforms/BaseField';
		import nothing    from 'uniforms/nothing';
		import {Children} from 'react';
		
		// We have to ensure that there's only one child, because returning an array
		// from a component is prohibited.
		const DisplayIf = ({children, condition}, {uniforms}) =>
		    condition(uniforms)
		        ? Children.only(children)
		        : nothing
		;
		
		DisplayIf.contextTypes = BaseField.contextTypes;
		export default DisplayIf;


----------

	const ThreeStepForm = ({schema}) =>
    <AutoForm schema={schema}>
        <TextField name="fieldA" />

        <DisplayIf condition={context => context.model.fieldA}>
            <section>
                <TextField name="fieldB" />

                <DisplayIf condition={context => context.model.fieldB}>
                    <span>
                        Well done!
                    </span>
                </DisplayIf>
            </section>
        </DisplayIf>
    </AutoForm>
	;

- SubmitButton（Example2）

		import BaseField      from 'uniforms/BaseField';
		import React          from 'react';
		import filterDOMProps from 'uniforms/filterDOMProps';
		
		// This field works as follows: render standard submit field and disable it, when
		// the form is invalid. It's a simplified version of a default SubmitField from
		// uniforms-unstyled.
		const SubmitField = (props, {uniforms: {error, state: {disabled}}}) =>
		    <input disabled={!!(error || disabled)} type="submit" />
		;
		
		SubmitField.contextTypes = BaseField.contextTypes;
		
		export default SubmitField;

- SwapField（Example3）交换值

		import BaseField      from 'uniforms/BaseField';
		import get            from 'lodash/get';
		import {Children}     from 'react';
		import {cloneElement} from 'react';
		
		// This field works as follows: on click of its child it swaps values of fieldA
		// and fieldB. It's that simple.
		const SwapField = ({children, fieldA, fieldB}, {uniforms: {model, onChange}}) =>
		    cloneElement(Children.only(children), {
		        onClick () {
		            const valueA = get(model, fieldA);
		            const valueB = get(model, fieldB);
		
		            onChange(fieldA, valueB);
		            onChange(fieldB, valueA);
		        }
		    })
		;
		
		SwapField.contextTypes = BaseField.contextTypes;
		
		export default SwapField;


----------
	<section>
	    <TextField name="firstName" />
	    <SwapField fieldA="firstName" fieldB="lastName">
	        <Icon name="refresh" />
	    </SwapField>
	    <TextField name="lastName" />
	</section>

