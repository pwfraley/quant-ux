
<script>
import on from 'dojo/on'
import lang from 'dojo/_base/lang'
import css from 'dojo/css'
import win from 'dojo/_base/win'
import topic from 'dojo/topic'

export default {
    name: 'Tools',
    methods: {
		startEyeDropper (isShift, isCntrl) {
		
			if (!window.EyeDropper) {
				this.showError('Color Selection is not supported on your browser. Try Chrome?')
			}

			if (!this._selectWidget) {
				this.showHint('Select an element to use the color selection')
				return
			}
			const eyeDropper = new window.EyeDropper()
			eyeDropper.open().then(result => {
				const color = result.sRGBHex
				if (this._selectWidget.type === 'Label' || isShift) {
					this.controller.updateWidgetProperties(this._selectWidget.id, {color: color}, 'style');
					return
				} 
				
				if (isCntrl) {
					this.controller.updateWidgetProperties(this._selectWidget.id, { 
						borderTopColor: color, 
						borderBottomColor: color,
						borderRightColor: color,
						borderLeftColor: color
					}, 'style');
					return
				}
				
				this.controller.updateWidgetProperties(this._selectWidget.id, {background: color}, 'style');
				
			}).catch(e => {
				this.showError('Oooppps.. Something wenr wrong')
				console.error(e)
			})
			
		},

      	renderScreenDistance (){
			this.cleanUpAlignment();
			if(!this._alignmentToolInited){
				this.alignmentStart("widget", this._selectWidget, "All");
				this._alignmentTool.showScreenDistance(this._selectWidget);
			}
		},

		/**
		 * Called on mouse over!
		 */
		renderWidgetDistance (widget) {
			/**
			 * In case no widget was selected we init with the hover one
			 */
			if (!this._alignmentToolInited && widget){
				this.alignmentStart("widget", widget, "All");
			}
			if(this._selectWidget && widget && this._selectWidget.id != widget.id){
				this._alignmentTool.showWidgetDistance(this._selectWidget, widget);
			} else if(this._selectWidget){
				this._alignmentTool.showScreenDistance(this._selectWidget);
			} else if(widget){
				this._alignmentTool.showScreenDistance(widget);
			}
			this.setHoverWidget(widget);
		},

		/**********************************************************************
		 * Box Tool
		 **********************************************************************/


		onToolTextStart (e){

			this.stopEvent(e);
			this.cleanUpSelectionListener();
			this.unSelect();

			this.alignmentStart("grid");

			this._selectionToolStart = this.getCanvasMousePosition(e);
			this._selectionToolStart = this.allignPosition(this._selectionToolStart, e)

			this._selectionToolMoveListener = on(win.body(),"mousemove", lang.hitch(this,"onTooltMove", "MatcAddTool"));
			this._selectionToolUpListener = on(win.body(),"mouseup", lang.hitch(this,"onToolTextEnd"));

			topic.publish("matc/canvas/click", "");

			return false;
		},

		onToolTextEnd (e){
			this.logger.log(0,"onToolTextEnd", "enter > ");
			this.stopEvent(e);

			var pos = this._getSelectionToolBox();
			let noBox = false
			if (!pos) {
				/**
				 * If we did not draw a box, we create a default one
				 */
				this.logger.log(0,"onToolTextEnd", "new mode > ");
				pos = this.getCanvasMousePosition(e);
				pos.w = this.getZoomed(1000, this.zoom);
				pos.h = this.getZoomed(20, this.zoom);
				var lastText = this.controller.getLastChangedWidget("Label");
				if (lastText) {
					pos.h = this.getZoomed(lastText.h, this.zoom);
				}
				noBox = true
			}

			this._addText(pos, noBox);

			return false;
		},

		onToolTextCreateAsPos (e){
			this.logger.log(0,"onToolTextCreateAsPos", "enter > ");
			this.stopEvent(e);

			var pos = this.getCanvasMousePosition(e);
			pos.w = this.getZoomed(200, this.zoom);
			pos.h = this.getZoomed(20, this.zoom);
			this._addText(pos);

			return false;
		},


		_addText (pos, noBox = false){
			this.cleanUpSelectionListener();
			this.controller.setMode("edit", true);

			if (pos){
				var widget = {
					"type" : "Label",
					"name" : "Label",
					"x" : pos.x,
					"y" : pos.y,
					"w" : pos.w,
					"h" : pos.h,
					"z" : 0,
					"props" : {
						"label" : noBox ? "Type something" : ''
					},
					"has" : {
						"label" : true,
						"padding" : true,
						"advancedText" : true
					},
					"style" : {
						"fontSize" : "Auto",
						"fontFamily" : "Helvetica Neue,Helvetica,Arial,sans-serif",
						"textAlign" : "left",
						"letterSpacing" : 0,
						"lineHeight" : 1,
						"color": "#333333",
						"textShadow": null
					}
				};

				if (this.controller){
					var lastText = this.controller.getLastChangedWidget("Label");
					if (lastText && lastText.style){
						if (lastText.style.color){
							widget.style.color = lastText.style.color;
						}
						if (lastText.style.fontFamily){
							widget.style.fontFamily = lastText.style.fontFamily;
						}
					}
				}

				var newWidget = this.controller.addWidget(widget, pos, true);
				if (newWidget) {

					if (noBox !== true) {
						this.onWidgetSelected(newWidget.id, true);
					} else {
						/**
						 * Make sure the DND div is hidden as fast as possible.
						 */
						this.addAfterRenderCallBack(() => {
							this.selectDnDBox(newWidget.id)
						}, true)
					}

					setTimeout(() => {
						this.inlineEditInit(newWidget, noBox);
					},150);
				}

			} else {
				this.logger.warn("_addText", "no pos passed");
			}
		},

		/**********************************************************************
		 * Text Tool
		 **********************************************************************/

		onToolBoxInit (){
			/**
			 * To make the highlighting work before starting...
			 *
			 * 1) init tool to have mouse move listener
			 *
			 * 2) call alignPosition to make render
			 *
			 * 3) on clikc onToolbaxStart is called. To not set selectionToolStart to current mouse pointer
			 * but to selectionToolInit
			 *
			 * FIXME: This does not work when called with kezboard
			 */
			//this.onTooltInit("MatcAddTool");
		},

		onToolBoxStart (e){


			this.stopEvent(e);
			this.unSelect();
			this.cleanUpSelectionListener();


			this.alignmentStart("grid", null, null, null, true);

			this._selectionToolStart = this.getCanvasMousePosition(e);
			this._selectionToolStart = this.allignPosition(this._selectionToolStart, e)
			
			if (this._alignmentTool) {
				this._alignmentTool.startPos = this._selectionToolStart
				this._alignmentTool.allowShift = true
			}
			
			

			this.initToolMouseMoveListener("MatcAddTool");
			this._selectionToolUpListener = on(win.body(),"mouseup", lang.hitch(this,"onToolBoxEnd"));

			topic.publish("matc/canvas/click", "");

			return false;

		},

		onToolBoxEnd (e){
			this.logger.log(0,"onToolBoxEnd", "enter > ");
			this.stopEvent(e);

			var pos = this._getSelectionToolBox();

			this.cleanUpSelectionListener();
			this.controller.setMode("edit", false);

			if (pos)	{

				var widget = {
					"type" : "Button",
					"name" : "Box",
					"x" : pos.x,
						"y" : pos.y,
						"w" : pos.w,
						"h" : pos.h,
						"z" : 0,
						"props" : {
							"label" : ""
						},
						"has" : {
								"backgroundColor" : true,
								"border" : true,
								"label" : true,
								"padding" : true,
								"onclick" : true
						},
						"actions":{},
						"style" : {
							"fontSize" : 14,
							"fontFamily" : "Helvetica Neue,Helvetica,Arial,sans-serif",
							"textAlign" : "left",
							"letterSpacing" : 0,
							"lineHeight" : 1.4,
							"color" : "#ffffff",
							"borderTopRightRadius" : 0,
								"borderTopLeftRadius" : 0,
								"borderBottomRightRadius" : 0,
								"borderBottomLeftRadius" : 0,
								"borderTopWidth" : 0,
								"borderBottomWidth" : 0,
								"borderRightWidth" : 0,
								"borderLeftWidth" : 0,
								"borderTopColor" : "#333333",
								"borderBottomColor" : "#333333",
								"borderRightColor" : "#333333",
								"borderLeftColor" : "#333333",
								"background" : "#333333",
								"paddingTop" : 0,
								"paddingBottom" : 0,
								"paddingLeft" : 0,
								"paddingRight" : 0,
								"textShadow": null
						}
				};


				if (this.controller){
					var lastText = this.controller.getLastChangedWidget("Button");
					if (lastText && lastText.style){
						if (lastText.style.background){
							widget.style.background = lastText.style.background;
						}
						if (lastText.style.color){
							widget.style.color = lastText.style.color;
						}
						if (lastText.style.fontFamily){
							widget.style.fontFamily = lastText.style.fontFamily;
						}
					}
				}

				var newWidget = this.controller.addWidget(widget, pos, true);
				if(newWidget){
					requestAnimationFrame( ()=> {
						this.onWidgetSelected(newWidget.id, true);
					})
				}
			}else {
				this.logger.warn("onToolBoxEnd", "no pos passed");
			}

			return false;
		},


		/**********************************************************************
		 * HotSpot
		 **********************************************************************/


		onToolHotspotStart (e){
			this.stopEvent(e);
			this.unSelect();
			this.cleanUpSelectionListener();

			this.alignmentStart("grid");

			this._selectionToolStart = this.allignPosition(this.getCanvasMousePosition(e), e);
			if (this._alignmentTool) {
				this._alignmentTool.startPos = this._selectionToolStart
				this._alignmentTool.allowShift = true
			}
			

			this._selectionToolMoveListener = on(win.body(),"mousemove", lang.hitch(this,"onTooltMove", "MatcHotspotTool"));
			this._selectionToolUpListener = on(win.body(),"mouseup", lang.hitch(this,"onToolHotspotEnd"));

			topic.publish("matc/canvas/click", "");
			return false;
		},

		onToolHotspotEnd (e){
			this.logger.log(0,"onToolHotspotEnd", "enter > ");
			this.stopEvent(e);

			var pos = this._getSelectionToolBox();

			this.cleanUpSelectionListener();
			this.controller.setMode("edit", false);

			if(pos){
				var widget = {
					"type" : "HotSpot",
					"name" : "Hotspot",
					"x" : pos.x,
					"y" : pos.y,
					"w" : pos.w,
					"h" : pos.h,
					"z" : 0,
					"props" : {},
					"has" : {
							"onclick" : true
					},
					"actions":{},
					"style" : {}
				};
				var newWidget = this.controller.addWidget(widget, pos, true);
				if(newWidget){
					requestAnimationFrame( () => {
						this.onWidgetSelected(newWidget.id, true);
					})
				}
			} else {
				this.logger.warn("onToolHotspotEnd", "no pos passed");
			}
			return false;
		},


		/**********************************************************************
		 * Generic Move tool:
		 * FIXME: Selection should use the same
		 **********************************************************************/
		onTooltInit (type){
			this.logger.log(-1,"onTooltInit", "enter > ");

			/**
			 * create dummy model
			 */
			this.initToolMouseMoveListener(type);

			var temp = {
				x:0,
				y:0,
				w:1,
				h:1,
				id: "temp"
			}
			this.alignmentStart("widget", temp, "All", null, false);

		},


		initToolMouseMoveListener (type){
			if (!this._selectionToolMoveListener){
				this.logger.log(-1,"initToolMouseMoveListener", "Create listener ");
				this._selectionToolMoveListener = on(win.body(),"mousemove", lang.hitch(this,"onTooltMove", type));
			}
		},

		onTooltMove (css, e){
			this.logger.log(3,"onTooltMove", "enter > ");
			this.stopEvent(e);

			if(this._selectionToolStart){
				this._selectionToolEnd = this.getCanvasMousePosition(e);
				this._selectionToolEnd = this.allignPosition(this._selectionToolEnd, e);

				if(!window.requestAnimationFrame){
					console.warn("No requestAnimationFrame()");
					this._renderToolMove(css);
				} else {
					const callback = lang.hitch(this, "_renderToolMove",css);
					requestAnimationFrame(callback);
				}
			} else {
				const pos = this.getCanvasMousePosition(e);
				pos.w = 1;
				pos.h = 1;
				this._selectionToolInit = this.allignPosition(pos, e);
			}

			return false;

		},

		/**
		 * FIXME: Merge latest for *box add* with selction to general drawing tool. Just make
		 * this css  (MatcHotspotTool) interchangeable
		 */
		_renderToolMove (cssClass){

			if(!this._selectionToolDiv){
				this._selectionToolDiv = document.createElement("div");
				css.add(this._selectionToolDiv,cssClass);
				this.dndContainer.appendChild(this._selectionToolDiv);
			}

			var pos = this._getSelectionToolBox();

			if(pos){
				this._selectionToolDiv.style.top = pos.y + "px";
				this._selectionToolDiv.style.left = pos.x + "px";

				this._selectionToolDiv.style.width = pos.w + "px";
				this._selectionToolDiv.style.height = pos.h + "px";
			}


		},



		/**********************************************************************
		 * Arrow Key
		 **********************************************************************/


		onArrowLeft (e, isShift){
			this.logger.log(0,"onArrowLeft", "enter > "+ e.altKey);
			this.controller.incMultiWidgetPosition(this.getSelectedIds(), -1 * this.getArrowMoveDistance(isShift, true), 0);
		},

		onArrowRight (e, isShift){
			this.logger.log(0,"onArrowRight", "enter > " + e.altKey);
			this.controller.incMultiWidgetPosition(this.getSelectedIds(), 1 * this.getArrowMoveDistance(isShift, true), 0);
		},

		onArrowUp (e, isShift){
			this.logger.log(0,"onArrowUp", "enter > " + e.altKey);
			this.controller.incMultiWidgetPosition(this.getSelectedIds(), 0, -1 * this.getArrowMoveDistance(isShift, false));
		},

		onArrowDown (e, isShift){
			this.logger.log(0,"onArrowDown", "enter > "+ e.altKey);
			this.controller.incMultiWidgetPosition(this.getSelectedIds(), 0, 1 * this.getArrowMoveDistance(isShift, false));
		},

		getArrowMoveDistance (isShift, isHorizontal) {
			if (isShift) {
				if (this.model && this.model.grid) {
					if (isHorizontal) {
						return this.model.grid.w
					} else {
						return this.model.grid.h
					}
				}
			}
			return 1
		},

		/**********************************************************************
		 * Group
		 **********************************************************************/


		onGroup (){
			this.logger.log(3,"onGroup", "enter > ");

			if(this._selectGroup){
				this.logger.log(0,"onGroup", "Remove group " + this._selectGroup.id);
				this.controller.removeGroup(this._selectGroup.id);
			} else if(this._selectMulti){
				this.logger.log(0,"onGroup", "Create Group");
				var group = this.controller.addGroup(this._selectMulti);
				if(group){
					this.onGroupSelected(group.id);
				}
			} else {
				this.showHint("Select a some widget to group!");
			}
		},


		/**********************************************************************
		 * Alignment
		 **********************************************************************/

		onAlignStart (direction){
			this.logger.log(0,"onAlignStart", "enter > "+ direction);

			/**
			 * We can only copy the style of an widget
			 */
			if(this._selectWidget || this._selectGroup ){

				this._alignSource = [];
				this.alignDirection = direction;
				if(this._selectWidget){
					this._alignSource.push(this._selectWidget.id);
				}
				if(this._selectGroup){
					this._alignSource = this._selectGroup.children;
				}

				this.setState(10);

				this.setBoxClickCallback("onAlignEnd");
				this.setCanvasClickCallback("onAlignCancel");
				this.setCanvasCancelCallback("onAlignCancel");

				this.showHint("Select the widget or group you want to align to");
			}

		},

		onAlignEnd (id, div){
			this.logger.log(0,"onAlignEnd", "enter > " +  id, div);

			if(this._alignSource){
				var group = this.getParentGroup(id);
				if(this._selectGroup){
					if(group){
						this.controller.alignGroup(this.alignDirection, this._alignSource, group.children);
					} else {
						this.controller.alignGroup(this.alignDirection, this._alignSource, [id]);
					}
				} else {
					if(group){
						this.controller.alignWidgets(this.alignDirection, this._alignSource, group.children);
					} else {
						this.controller.alignWidgets(this.alignDirection, this._alignSource, [id]);
					}
				}

			}
			this.onAlignCancel();
		},

		onAlignCancel (){
			this.logger.log(0,"onAlignCancel", "enter");
			this._alignSource = null;
			this.alignDirection = null;
			this.cleanUpClickCallbacks();
			this.cleanUpCancelCallbacks();
			this.controller.renderAlignEnd();
			this.setState(0);
		},

		/**********************************************************************
		 * Selection Tool
		 **********************************************************************/



		onSelectionStarted (e){
			this.logger.log(2,"onSelectionStarted", "enter > ");

			/**
			 * In case something is added (screen, widht or comment) we do not
			 * want to render selections
			 */
			if (this.state != 3 && this.controller){
				this.controller.setMode("select");
				this.stopEvent(e);
				if(!this._selectionToolStart){
					this.unSelect();
					this._selectionToolStart = this.getCanvasMousePosition(e);
					this._selectionToolMoveListener = on(win.body(),"mousemove", lang.hitch(this,"onSelectionMove"));
					this._selectionToolUpListener = on(win.body(),"mouseup", lang.hitch(this,"onSelectionEnd"));
				}
			}
			topic.publish("matc/canvas/click", "");
			return false;
		},

		onSelectionMove (e){
			//this.logger.log(4,"onSelectionMove", "enter > ");
			this.stopEvent(e);
			if(this._selectionToolStart){
				this._selectionToolEnd = this.getCanvasMousePosition(e);
				if(!window.requestAnimationFrame){
					console.warn("No requestAnimationFrame()");
					this._renderSelectionTool();
				} else {
					var callback = lang.hitch(this, "_renderSelectionTool");
					requestAnimationFrame(callback);
				}
			}
			return false;
		},

		_renderSelectionTool (){

			if(!this._selectionToolDiv){
				this._selectionToolDiv = document.createElement("div");
				css.add(this._selectionToolDiv, "MatcSelectionTool");
				this.dndContainer.appendChild(this._selectionToolDiv);
			}
			var pos = this._getSelectionToolBox();
			if(pos){
				this._selectionToolDiv.style.top = pos.y + "px";
				this._selectionToolDiv.style.left = pos.x + "px";
				this._selectionToolDiv.style.width = pos.w + "px";
				this._selectionToolDiv.style.height = pos.h + "px";
			}
		},

		_getSelectionToolBox (){

			/**
			 * FIXME: This can be null sometimes
			 */
			if(this._selectionToolStart== null || this._selectionToolEnd == null){
				this.controller.setMode("edit");
				this.cleanUpSelectionListener();
				return null;
			}

			var pos = {x:0, y:0, w: 5, h : 5};

			if(this._selectionToolStart.x < this._selectionToolEnd.x){
				pos.x = this._selectionToolStart.x;
			} else {
				pos.x = this._selectionToolEnd.x;
			}

			if(this._selectionToolStart.y < this._selectionToolEnd.y){
				pos.y = this._selectionToolStart.y;
			} else {
				pos.y = this._selectionToolEnd.y;
			}

			pos.w = Math.abs(this._selectionToolStart.x - this._selectionToolEnd.x);
			pos.h = Math.abs(this._selectionToolStart.y - this._selectionToolEnd.y);
			return pos;
		},

		onSelectionEnd (e){
			this.logger.log(2,"onSelectionEnd", "enter > ");
			this.stopEvent(e);

			var selection = [];
			var selectedScreens = [];
			var pos = this._getSelectionToolBox();

			if(pos && pos.w > 3 && pos.h > 3 ){
				/**
				 * Check Screens
				 */
				for(let id in this.model.screens){
					let screen = this.model.screens[id];
					if(this._isContainedInBox(screen, pos)){
						selectedScreens.push(screen);
					}
				}

				/**
				 * Check widgets
				 */
				let topGroups = {}
				let elementsWidthGroup = []
				for(let id in this.model.widgets){
					let widget = this.model.widgets[id];
					if(!widget.inherited){
						if(this._isContainedInBox(widget, pos)){
							if (selection.indexOf(widget.id) < 0) {
								selection.push(widget.id);
							}

							/**
							 * Sincen 2.1.3 we also extand groups
							 */
							selection = this.extendSelectionToGroup(widget.id, selection)

							let topGroup = this.getTopParentGroup(id)
							if (topGroup) {
								topGroups[topGroup.id] = topGroup
							} else {
								elementsWidthGroup.push(id)
							}
						}
					}
				}


				/**
				 * If there is a multi selection, save at and
				 * set mode back to edit! if only one is selected
				 * set at selected Widget.
				 *
				 * FIXME: Call renderSelection instead passing forceRender=true!
				 *
				 * Since 2.1.3 we have subgroups. If all elements have the save top group
				 * we select the group instead
				 */
				topGroups = Object.values(topGroups)
				if (topGroups.length === 1 && elementsWidthGroup.length === 0) {
					this._selectGroup = topGroups[0];
					this.controller.setMode("edit", false);
				} else if(selectedScreens.length >= 1){
					this._selectedScreen = selectedScreens[0];
					this.controller.setMode("edit", false);
				} else if(selection.length > 1){
					this._selectMulti = selection;
					this.controller.setMode("edit", false);
				} else if(selection.length == 1){
					let id = selection[0];
					this._selectWidget = this.model.widgets[id];
					this.controller.setMode("edit", false);
				} else {
					this.onCanvasSelected();
					this.controller.setMode("edit", false);
				}

			} else {

				/**
				 * FIXME: Check here for line clicks to, as the user did nothing...
				 */
				this.onCanvasSelected();
				this.controller.setMode("edit");
			}

			/**
			 * call setMode()??
			 */
			this.cleanUpSelectionListener();
			return false;
		},

		cleanUpSelectionListener (){
			this.logger.log(5,"cleanUpSelectionListener", "enter > ");

			if (this._selectionToolMoveListener){
				this._selectionToolMoveListener.remove();
				this._selectionToolMoveListener = null;
			}

			if (this._selectionToolUpListener){
				this._selectionToolUpListener.remove();
				this._selectionToolUpListener = null;
			}

			this._selectionToolStart = null;
			this._selectionToolEnd = null;
			this._selectionToolInit = null;

			if (this._selectionToolDiv && this._selectionToolDiv.parentNode){
				this._selectionToolDiv.parentNode.removeChild(this._selectionToolDiv);
				this._selectionToolDiv = null;
			}

		},

		/**********************************************************************
		 * Copy Style
		 **********************************************************************/

		onCopyStyle (){
			this.logger.log(0,"onCopyStyle", "enter > " + this._selectWidget);

			/**
			 * We can only copy the style of an widget
			 */
			if(this._selectWidget || this._selectedScreen ){

				if(this._selectWidget){
					this._copiedStyle = this._selectWidget.id;
				}

				if( this._selectedScreen){
					this._copiedStyle = this._selectedScreen.id;
				}

				this.setBoxClickCallback("onPasteStyle");
				this.setCanvasClickCallback("cleanUpCopyPasteStyle");
				this.setCanvasCancelCallback("cleanUpCopyPasteStyle");
			}
		},

		onPasteStyle (id){
			this.logger.log(4,"onPasteStyle", "enter  > " + id);
			if(this._copiedStyle){
				this.controller.onCopyWidgetStyle(this._copiedStyle, id);
				this.onWidgetSelected(id);
			} else {
				console.error("onPasteStyle() called altough no style is copied");
			}
			this.cleanUpCopyPasteStyle();
		},

		cleanUpCopyPasteStyle (){
			this.logger.log(0,"cleanUpCopyPasteStyle", "enter");
			this._copiedStyle = null;
			this.cleanUpClickCallbacks();
			this.cleanUpCancelCallbacks();
			this.controller.renderCopyPasteStyleEnd();
			this.setState(0);
		},

		/**********************************************************************
		 * Copy Paste!
		 **********************************************************************/

		onCopy (isDuplicate){
			this.logger.log(-1,"onCopy", "enter > " + isDuplicate);

			if(this._selectWidget || this._selectedScreen || this._selectMulti || this._selectGroup){
				this._copied ={
					widget : this._selectWidget,
					screen : this._selectedScreen,
					multi : this._selectMulti,
					group  :this._selectGroup
				};

				this.toolbar.showCopyPaste();
				// TODO: Store in cookie / local db or so!
				this._setClipBoard();
			}

			if(this._selectWidget){
				this.showSuccess("The widget was copied!");
			} else if(this._selectedScreen){
				this.showSuccess("The screen was copied!");
			} else if(this._selectMulti || this._selectGroup){
				this.showSuccess("The widgets were copied!");
			} else {
				this.showHint("Nothing selected to copy");
			}

			/**
			 * If we have a fresh copy we have to reset
			 * the lastPaste stuff
			 */
			if (!isDuplicate){
				delete this._lastPaste
			}

		},

		_setClipBoard (){
			this.logger.log(-1,"_setClipBoard", "enter > ", this._selectedScreen);
			this.controller.setClipBoard (this._selectWidget, this._selectedScreen, this._selectMulti, this._selectGroup)
		},


		_getClipBoard (){
			this.logger.log(-1,"_setCligBoard", "enter > ");
			return this.controller.getClipBoard ()
		},

		hasCopy (){
			return this._copied;
		},

		onPaste (fromToolBar, e){

			/**
			 * Since 2.2.6 we support the clipboard
			 * To ensure this works also with the patterns,
			 * we just use this for the if the clipboard comes
			 * from another app.
			 *
			 * TODO: We could unify this...
			 */
			let clipBoard = this._getClipBoard();
			let pos = this.getLastMousePos();
			if (clipBoard && clipBoard.id !== this.model.id) {

				this.logger.log(-1,"onPaste", "enter > OTHER APP");
				if (!fromToolBar) {
					this.unSelect();

					this.controller.onPasteClipBoard(clipBoard, pos);
					this.showSuccess("Clipboard was pasted!");
				} else {
					/**
					 * TODO: Use the bounding nbox to also have DND
					 */
					this.showError("Copies from other apps work only with CTRL-V");
				}

			} else if (this._copied){
				this.logger.log(-1,"onPaste", "enter > SAME APP  > "+  fromToolBar);

				if (!fromToolBar){

					/**
					 * If paste was not triggered by toolbar, simply add.
					 * Otherwise render preview and wait for click!
					 */
					this.unSelect();
					pos = this.getPastePostion(pos);

					/**
					 * Store last paste for pattern paste
					 */
					var lastPaste = {
						ts: new Date().getTime(),
						source: this._copied,
						target: {}
					};
					/**
					 * If we have multiple ctrl-v the this.copied stays the same
					 * so the distance will be calculated always to the first element,
					 * and will this increase.
					 * We have to therefore update the source to the last paste
					 */
					if(this._lastPaste && this._lastPaste.target){
						lastPaste.source = this._lastPaste.target;
					}

					if (this._copied.widget){
						// TODO: Add a pasteClipBoardMethod, which would somehow to return a copy...
						var copy = this.controller.onCopyWidget(this._copied.widget.id, pos);
						if(copy){
							requestAnimationFrame( () => {
								this.onWidgetSelected(copy.id, true);
							})
							lastPaste.target.widget = copy
						}
						this.showSuccess("Widget was pasted!");
					} else if(this._copied.screen){
						let newScreen = this.controller.onCopyScreen(this._copied.screen.id, pos);
						if (newScreen) {
							requestAnimationFrame( () => {
								this.onScreenSelected(newScreen.id, true);
							})
						}
						this.showSuccess("Screen was pasted!");
					} else if(this._copied.multi){
						this._selectMulti = this.controller.onMultiCopyWidget(this._copied.multi,pos);
						this.showSuccess("Widgets were pasted!");
						lastPaste.target.multi = this._selectMulti;
					} else if(this._copied.group){
						let copy = this.controller.onCopyGroup(this._copied.group, pos);
						if (copy) {
							this.showSuccess("Group were pasted!");
							requestAnimationFrame( () => {
									this.onGroupSelected(copy.id, true);
							})
							lastPaste.target.group = copy;
						}

					}

					if (!pos.newScreen){
						this.logger.log(3,"onPaste", "exit > store lastPaste");
						this._lastPaste = lastPaste;
					} else {
						this.logger.log(3,"onPaste", "exit > New Screen");
						delete this._lastPaste
					}

				} else {

					/**
					 * We were triggered from toolbar, and the user might want to still
					 * move the stuff around...
					 */

					if(this._copied.widget){
						/**
						 * FIXME! We should get here the widget from the source and copy that
						 */
						let widget = this.getZoomedSourceWidget(this._copied.widget)
						widget.id = "_temp";
						let div = this.createZoomedWidget(widget);
						if(!this._alignmentToolInited){
							this.alignmentStart("widget", widget, "All");
						}
						this._onAddNDropStart(div, widget, e, "onWidgetPaste");
						this.setState(3);
					}

					if(this._copied.screen){
						let screen = lang.clone(this._copied.screen);
						screen.id="_temp";
						screen.x = 0
						screen.y = 0
						let div = this.createScreen(screen);
						if(!this._alignmentToolInited){
							this.alignmentStart("screen", screen, "All");
						}
						this._onAddNDropStart(div, screen, e, "onScreenPaste");
						this.setState(3);
					}

					if(this._copied.multi){

						let selection = this._copied.multi;
						let boundingBox = this.getBoundingBox(selection);
						let div = this.createBox(boundingBox);

						// add in right order!
						let widgets = [];
						for(let i=0; i < selection.length; i++){
							let id = selection[i];
							let widget = this.model.widgets[id];
							widgets.push(widget);
						}
						widgets = this.getOrderedWidgets(widgets);

						for (let i=0; i< widgets.length; i++){
							let widget = widgets[i];
							let cloned = lang.clone(widget);
							cloned.id="_temp"+i;
							// add children relative to bounding box!
							cloned.x =  cloned.x - boundingBox.x;
							cloned.y =  cloned.y - boundingBox.y;
							let clonedDiv = this.createZoomedWidget(cloned);
							div.appendChild(clonedDiv);
						}

						if(!this._alignmentToolInited){
							this.alignmentStart("boundingbox", boundingBox, "All");
						}

						this._onAddNDropStart(div, this._copied.multi, e, "onMultiPaste");
						this.setState(3);
					}

					if(this._copied.group){
						let group = this._copied.group;
						let boundingBox = this.getBoundingBox(group.children);
						let div = this.createBox(boundingBox);

						// add in right order!
						var widgets = [];
						for(let i=0; i < group.children.length; i++){
							let id = group.children[i];
							let widget = this.model.widgets[id];
							widgets.push(widget);
						}
						widgets = this.getOrderedWidgets(widgets);

						for (let i=0; i< widgets.length; i++){
							let widget = widgets[i];
							let cloned = lang.clone(widget);
							cloned.id="_temp"+i;
							cloned.x =  cloned.x - boundingBox.x;
							cloned.y =  cloned.y - boundingBox.y;
							let clonedDiv = this.createWidget(cloned);
							div.appendChild(clonedDiv);
						}

						if(!this._alignmentToolInited){
							this.alignmentStart("boundingbox", boundingBox, "All");
						}

						this._onAddNDropStart(div, group, e, "onGroupPaste");
						this.setState(3);
					}
				}

			}
		},

		getZoomedSourceWidget (widget) {
			if (this.sourceModel && this.sourceModel.widgets[widget.id]) {
				var z = this.getZoomFactor();
				let sourceWidget = lang.clone(this.sourceModel.widgets[widget.id])
				return this.getZoomedBox(sourceWidget,z,z);
			}
			return widget
		},

		getPastePostion (mousePos){
			/**
			 * If we do not clone here, the controller might zoom the mousePosition...
			 */
			let pos = lang.clone(mousePos)
			let newScreen = this.getHoverScreen(pos);

			let oldScreen;
			let boundingBox;
			if (this._copied.widget){
				oldScreen = this.getHoverScreen(this._copied.widget);
			}
			if (this._copied.multi){
				boundingBox = this.getBoundingBox(this._copied.multi)
				oldScreen = this.getHoverScreen(boundingBox);
			}
			if(this._copied.group){
				boundingBox = this.getBoundingBox(this._copied.group.children);
				oldScreen = this.getHoverScreen(boundingBox);
			}



			if (oldScreen && newScreen && oldScreen.id != newScreen.id) {
				/**
			 * If we paste to new screen, we do not want have the last paste
			 * function active! We paste at same postion if it fits, otherwise
				 * teh current mouse position
				 */
				pos = this.getPastePositionInScreen(pos, oldScreen, newScreen, boundingBox)


			} else {
				/**
				 * Paste on same screen. Now check if we have a paste pattern...
				 */
				if (this._lastPaste ){
					var now = new Date().getTime()
					if (now - this._lastPaste.ts < 20000) {

							var targetBBX = this.getPasteBoundingBox(this._lastPaste.target);
							var sourceBBX = this.getPasteBoundingBox(this._lastPaste.source);

							if(targetBBX && sourceBBX){

								var targetScreen = this.getHoverScreen(targetBBX);
								var sourceScreen = this.getHoverScreen(sourceBBX);

								if (targetScreen && sourceScreen && sourceScreen.id === targetScreen.id){
									var difX = targetBBX.x - sourceBBX.x;
									var difY = targetBBX.y - sourceBBX.y;
									/**
									 * FIXME: In some situations it can happen, that we would paste
									 * super far from the current point of view. In such a case both difs
									 * are large. We do not want that an stop.
									 */
									if (Math.abs(difY) < 5 || Math.abs(difX) < 5){
										return {
											x: targetBBX.x + difX,
											y: targetBBX.y + difY
										}
									} else {
										console.debug("Not aligned", difX, difY)
									}

								} else {
									console.debug("Pattern paste on new screen!")
								}
							}
					} else {
						console.debug("Timeout!");
					}
				}
			}
			return pos;
		},

		/**
		 * Check if we paste on new screen on same relative position
		 */
		getPastePositionInScreen (pos, oldScreen, newScreen, boundingBox) {

			/**
			 * FIXME: Check if the pos is in the visible part of the canvas! If not,
			 * return the pos.
			 *
			 */

			if (this._copied.widget){

				let x = (this._copied.widget.x - oldScreen.x) + newScreen.x;
				let y = (this._copied.widget.y - oldScreen.y) + newScreen.y;

				/**
				 * Only paste if we are in the screen
				 */
				if (y < (newScreen.y + newScreen.h)) {
					pos.newScreen = true
					pos.x = x
					pos.y = y
				} else {
					this.logger.log(-1,"getPastePositionInScreen()", "EXIT: Widget out of bounce");
				}
			}

			if (this._copied.multi || this._copied.group) {

				let x = (boundingBox.x - oldScreen.x) + newScreen.x;
				let y = (boundingBox.y - oldScreen.y) + newScreen.y;

				/**
				 * Only past if we are inn screen
				 */
				if (y < (newScreen.y + newScreen.h)) {
					pos.newScreen = true
					pos.x = x
					pos.y = y
				} else {
					this.logger.log(-1,"getPastePositionInScreen()", "EXIT: Group out of bounce");
				}
			}

			return pos
		},

		getPasteBoundingBox (copy){
			if(copy.widget){
				return this.getBoundingBox([copy.widget.id]);
			}
			if(copy.multi){
				return this.getBoundingBox(copy.multi);
			}
			if(copy.group){
				return this.getBoundingBox(copy.group.children);
			}
		},

		onMultiPaste (pos){
			if(this._copied){
				if(this._copied.multi){
					this._selectMulti = this.controller.onMultiCopyWidget(this._copied.multi,pos);
					this.showSuccess("Widgets were pasted!");
				}
			}
			this.setState(0);
		},

		onGroupPaste (pos){
			if(this._copied){
				if(this._copied.group){
					this._selectGroup = this.controller.onCopyGroup(this._copied.group, pos);
					this.showSuccess("Group was pasted!");
				}
				delete this._lastPaste;
			}
			this.setState(0);
		},

		onWidgetPaste (pos){
			if(this._copied){
				if(this._copied.widget){
					this.controller.onCopyWidget(this._copied.widget.id, pos);
					this.showSuccess("Widget was pasted!");
				}
				delete this._lastPaste;
			}
			this.setState(0);
		},

		onScreenPaste (pos){
			if(this._copied){
				if(this._copied.screen){
					this.controller.onCopyScreen(this._copied.screen.id, pos);
					this.showSuccess("Screen was pasted!");
				}
				delete this._lastPaste;
			}
			this.setState(0);
		},

		onDuplicate (){
			this.logger.log(0,"onDuplicate", "enter");
			// FIXME: Check if the last widget that was added was a copy if the selected one.
			// if so, check if the time was short (30 sec) and if was a pattern e.g
			this.onCopy(true);
			this.onPaste();
			delete this._copied;
		},


		onCut (){
			this.logger.log(0,"onCut", "enter");
		}
    },
    mounted () {
    }
}
</script>