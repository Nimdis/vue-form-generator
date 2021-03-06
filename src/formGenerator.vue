<template lang="jade">
	fieldset(v-if="schema != null")
		.form-group(v-for="field in fields", v-if="fieldVisible(field)", :class="getFieldRowClasses(field)")
			label {{ field.label }}
				span.help(v-if="field.help")
					i.fa.fa-question-circle
					.helpText {{{field.help}}}
			.field-wrap
				component(:is="getFieldType(field)", :disabled="fieldDisabled(field)", :model.sync="model", :schema.sync="field")
				.buttons(v-if="field.buttons && field.buttons.length > 0")
					button.btn.btn-default(v-for="btn in field.buttons", @click="btn.onclick(model, field)", :class="btn.classes") {{ btn.label }}
			.hint(v-if="field.hint") {{ field.hint }}
			.errors(v-if="field.errors && field.errors.length > 0")
				span(v-for="error in field.errors", track-by="$index") {{ error }}
</template>

<script>
	import Vue from "vue";
	//import Joi from "joi";
	import {each, isFunction, isNil, isArray, isString} from "lodash";

	// Load all fields from '../fields' folder
	let Fields = require.context("./fields/", false, /^\.\/field([\w-_]+)\.vue$/);
	let fieldComponents = {};
	each(Fields.keys(), (key) => {
		let compName = Vue.util.classify(key.replace(/^\.\//, "").replace(/\.vue/, ""));
		fieldComponents[compName] = Fields(key);
	});


	export default {
		components: fieldComponents,
		
		props: [
			"schema",
			"options",
			"model",
			"multiple",
			"isNewModel"
		],
		
		data () {
			return {
				errors: [] // Validation errors
			};
		},

		computed: {
			fields() {
				let res = [];
				if (this.schema) {
					each(this.schema.fields, (field) => {
						if (!this.multiple || field.multi === true)
							res.push(field);
					});
				}

				return res;
			}
		},

		watch: {
			// new model loaded
			model: function(newModel, oldModel) {
				if (oldModel == newModel) // model got a new property
					return;

				//console.log("Model changed!");
				if (this.options.validateAfterLoad === true && this.isNewModel !== true)
					this.validate();
				else
					this.clearValidationErrors();
			}
		},

		compiled() {
			// First load
			if (this.options && this.options.validateAfterLoad === true && this.isNewModel !== true)
				this.validate();
			else
				this.clearValidationErrors();
		},
	
		methods: {
			getFieldRowClasses(field) {
				let baseClasses = {
					error: field.errors && field.errors.length > 0, 
					disabled: this.fieldDisabled(field), 
					readonly: field.readonly, 
					featured: field.featured, 
					required: field.required
				};

				if (isArray(field.styleClasses)) {
					each(field.styleClasses, (c) => baseClasses[c] = true);
				}
				else if (isString(field.styleClasses)) {
					baseClasses[field.styleClasses] = true;
				}

				baseClasses["field-" + field.type] = true;

				return baseClasses;
			},

			getFieldType(fieldSchema) {
				return "field-" + fieldSchema.type;
			},

			fieldDisabled(field) {
				if (isFunction(field.disabled))
					return field.disabled(this.model);

				if (isNil(field.disabled))
					return false;

				return field.disabled;
			},

			fieldVisible(field) {
				if (isFunction(field.visible))
					return field.visible(this.model);

				if (isNil(field.visible))
					return true;

				return field.visible;
			},		

			validate() {
				this.clearValidationErrors();

				each(this.$children, (child) => {
					if (isFunction(child.validate))
					{
						let err = child.validate();
						each(err, (err) => {
							this.errors.push({
								field: child.schema,
								error: err
							});
						});
					}
				});

				return this.errors.length == 0;
			},

			clearValidationErrors() {
				this.errors.splice(0);

				each(this.$children, (child) => {
					child.clearValidationErrors();
				});				
			}
		}
	};
	
</script>

<style lang="sass">
	
	$errorColor: lighten(#F00, 0%);

	fieldset {
		
		input, select, textarea {
			border-radius: 4px;
			border: 1px solid #BBB;
			padding: 2px 5px;
		}
		
		span.help {
			margin-left: 0.3em;
			position: relative;

			.helpText {
				background-color: #444;
				bottom: 30px;
				color: #fff;
				display: block;
				left: 0px;
				//margin-bottom: 15px;
				opacity: 0;
				padding: 20px;
				pointer-events: none;
				position: absolute;
				text-align: justify;
				width: 300px;
				//transform: translateY(10%);
				transition: all .25s ease-out;
				box-shadow: 2px 2px 6px rgba(0, 0, 0, 0.5);
				border-radius: 6px;

				a {
					font-weight: bold;
					text-decoration: underline;
				}
			}

			/* This bridges the gap so you can mouse into the tooltip without it disappearing */
			.helpText:before {
				bottom: -20px;
				content: " ";
				display: block;
				height: 20px;
				left: 0;
				position: absolute;
				width: 100%;
			}  

			/* CSS Triangles - see Trevor's post */
			/*.helpText:after {
				border-left: solid transparent 10px;
				border-right: solid transparent 10px;
				border-top: solid #1496bb 10px;
				bottom: -10px;
				content: " ";
				height: 0;
				left: 50%;
				margin-left: -13px;
				position: absolute;
				width: 0;
			}*/
				
			&:hover .helpText {
				opacity: 1;
				pointer-events: auto;
				transform: translateY(0px);
			}					
		} // span.help

		.field-wrap {
			display: flex;

			.buttons {
				white-space: nowrap;
				button {
					display: inline-block;
					margin: 0 2px;
				}
			}
		}		

		.hint {
			font-style: italic;
			font-size: 0.8em;
		}


		.form-group {
			display: inline-block;
			vertical-align: top;
			width: 100%;
			// margin: 0.5rem 0.26rem;
			margin-bottom: 1rem;

			label {
				font-weight: 400;
			}

			&.featured {
				label {
					font-weight: bold;
				}			
			}

			&.required {
				label:after {
					content: "*";
					font-weight: normal;
					color: Red;
					position: absolute;
					padding-left: 0.2em;
					font-size: 1em;
				}	
			}

			&.disabled {
				label {
					color: #666;
					font-style: italic;
				}			
			}

			&.error {

				label {
					//color: $errorColor;
				}			

				input:not([type="checkbox"]), textarea, select {
					border: 1px solid $errorColor;
					background-color: rgba($errorColor, 0.15);
				}

				.errors {
					color: $errorColor;
					font-size: 0.80em;
					span {
						display: block;
						background-image: url('data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABAAAAAQCAYAAAAf8/9hAAAAiklEQVR4Xt2TMQoCQQxF3xdhu72MpZU3GU/meBFLOztPYrVWsQmEWSaMsIXgK8P8RyYkMjO2sAN+K9gTIAmDAlzoUzE7p4IFytvDCQWJKSStYB2efcAvqZFM0BcstMx5naSDYFzfLhh/4SmRM+6Agw/xIX0tKEDFufeDNRUc4XqLRz3qabVIf3BMHwl6Ktexn3nmAAAAAElFTkSuQmCC');
						background-repeat: no-repeat;
						padding-left: 17px;
							padding-top: 0px;
							margin-top: 0.2em;
							font-weight: 600;
					}

				} // .errors	

			} // .error

		} // .form-group

	} // fieldset
</style>