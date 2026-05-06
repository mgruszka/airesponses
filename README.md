# airesponses

Yes — here is the exact pattern for custom coach view -> boundary event -> human service flow -> service flow. IBM supports both calling a service from a coach view through a Service configuration option and using a named boundary event to send data from the coach to the human service flow, but only one boundary event can be specified per coach view. �
Architecture
Use this sequence:
User selects file in custom coach view.
Coach view reads the file as base64.
Coach view stores data in bound variables.
Coach view triggers boundary event onUploadRequested.
Human service catches that boundary event and calls a service flow.
Service flow calls your upload REST endpoint and returns result.
Human service updates variables and reloads the coach or moves forward. �
Because IBM notes that services called from coach views exchange JSON input and output, this pattern works best when the coach view passes serializable data such as file name, mime type, and base64 content rather than a browser-native File object. �
Data model
Create a BO named UploadData with these fields:
fileName : String
mimeType : String
fileSize : Integer
fileContentBase64 : String
uploadStatus : String
responseText : String
documentId : String
errorMessage : String �
In the client-side human service create:
tw.local.uploadData of type UploadData
tw.local.serviceResult of type UploadData or a separate response BO if you prefer. �
Service flow
Create a service flow named SF_UploadFile, because IBM requires the service flow to exist first before it can be referenced as a service configuration or called in the human-service logic. �
Inputs
Define input variable:
tw.local.uploadDataIn : UploadData �
Outputs
Define output variable:
tw.local.uploadDataOut : UploadData �
Inside service flow
Implement the logic:
Read tw.local.uploadDataIn.
Call your integration service or REST service.
Set tw.local.uploadDataOut.uploadStatus = "SUCCESS" or "ERROR".
Set returned values such as documentId, responseText, or errorMessage. �
Example server-side mapping idea:
tw.local.request = {
  fileName: tw.local.uploadDataIn.fileName,
    mimeType: tw.local.uploadDataIn.mimeType,
      fileSize: tw.local.uploadDataIn.fileSize,
        fileContentBase64: tw.local.uploadDataIn.fileContentBase64
        };
        Then your REST step posts that payload to the custom endpoint and maps the response back into tw.local.uploadDataOut. �
        Custom coach view
        Create a custom coach view named CV_FileUpload. On the Overview tab, define the single boundary event as onUploadRequested, because IBM states the boundary event is specified on the coach view and must later be wired in the human service flow. �
        Variables tab
        Create one binding:
        value of type UploadData �
        Create optional configuration options:
        maxSizeMb : Integer
        allowedTypes : String
        buttonLabel : String �
        Do not create the service flow as a service option inside the coach view for this boundary-event design, because in this pattern the service is called after the human service receives the event, not directly by the coach view. IBM supports direct service calls too, but here we are intentionally routing through the boundary event. �
        Coach view HTML
        <div class="fu-root">
          <div class="fu-row">
              <input type="file" class="fu-file" />
                  <button type="button" class="fu-btn">Prepare upload</button>
                    </div>
                      <div class="fu-name"></div>
                        <div class="fu-status"></div>
                        </div>
                        Coach view CSS
                        .fu-root {
                          display: flex;
                            flex-direction: column;
                              gap: 8px;
                              }

                              .fu-row {
                                display: flex;
                                  gap: 8px;
                                    align-items: center;
                                    }

                                    .fu-btn {
                                      padding: 6px 12px;
                                        cursor: pointer;
                                        }

                                        .fu-name,
                                        .fu-status {
                                          font-size: 12px;
                                          }
                                          Coach view JavaScript
                                          Put this in Behavior -> load:
                                          this.load = function() {
                                            var view = this;
                                              var root = this.context.element;
                                                var fileInput = root.find(".fu-file");
                                                  var button = root.find(".fu-btn");
                                                    var nameBox = root.find(".fu-name");
                                                      var statusBox = root.find(".fu-status");

                                                        var maxSizeMb = this.context.options.get("maxSizeMb") || 10;
                                                          var allowedTypes = this.context.options.get("allowedTypes") || "";
                                                            var buttonLabel = this.context.options.get("buttonLabel") || "Prepare upload";

                                                              button.text(buttonLabel);

                                                                if (allowedTypes) {
                                                                    fileInput.attr("accept", allowedTypes);
                                                                      }

                                                                        function setValue(prop, value) {
                                                                            view.context.binding.get("value").set(prop, value);
                                                                              }

                                                                                function getValue(prop) {
                                                                                    return view.context.binding.get("value").get(prop);
                                                                                      }

                                                                                        function setStatus(msg, code) {
                                                                                            statusBox.text(msg);
                                                                                                setValue("uploadStatus", code);
                                                                                                  }

                                                                                                    button.on("click", function() {
                                                                                                        var input = fileInput[0];
                                                                                                            var file = input && input.files && input.files.length ? input.files[0] : null;

                                                                                                                if (!file) {
                                                                                                                      setStatus("Please select a file.", "NO_FILE");
                                                                                                                            return;
                                                                                                                                }

                                                                                                                                    var maxBytes = maxSizeMb * 1024 * 1024;
                                                                                                                                        if (file.size > maxBytes) {
                                                                                                                                              setStatus("File exceeds " + maxSizeMb + " MB.", "TOO_LARGE");
                                                                                                                                                    return;
                                                                                                                                                        }

                                                                                                                                                            var reader = new FileReader();

                                                                                                                                                                reader.onload = function(e) {
                                                                                                                                                                      var dataUrl = e.target.result;
                                                                                                                                                                            var base64 = dataUrl.split(",")[1];

                                                                                                                                                                                  setValue("fileName", file.name);
                                                                                                                                                                                        setValue("mimeType", file.type || "application/octet-stream");
                                                                                                                                                                                              setValue("fileSize", file.size);
                                                                                                                                                                                                    setValue("fileContentBase64", base64);
                                                                                                                                                                                                          setValue("responseText", "");
                                                                                                                                                                                                                setValue("documentId", "");
                                                                                                                                                                                                                      setValue("errorMessage", "");
                                                                                                                                                                                                                            setStatus("File prepared. Sending to human service...", "READY_FOR_UPLOAD");

                                                                                                                                                                                                                                  nameBox.text("Selected: " + file.name);

                                                                                                                                                                                                                                        view.context.trigger("onUploadRequested");
                                                                                                                                                                                                                                            };

                                                                                                                                                                                                                                                reader.onerror = function() {
                                                                                                                                                                                                                                                      setValue("errorMessage", "Failed to read file.");
                                                                                                                                                                                                                                                            setStatus("Failed to read file.", "READ_ERROR");
                                                                                                                                                                                                                                                                };

                                                                                                                                                                                                                                                                    reader.readAsDataURL(file);
                                                                                                                                                                                                                                                                      });
                                                                                                                                                                                                                                                                      };
                                                                                                                                                                                                                                                                      This follows IBM’s documented model: the coach view writes JSON-serializable data to its binding and explicitly fires the named boundary event with this.context.trigger(...). �
                                                                                                                                                                                                                                                                      Human service wiring
                                                                                                                                                                                                                                                                      In your client-side human service:
                                                                                                                                                                                                                                                                      Add a coach.
                                                                                                                                                                                                                                                                      Put CV_FileUpload on the coach.
                                                                                                                                                                                                                                                                      Bind value to tw.local.uploadData.
                                                                                                                                                                                                                                                                      Configure options such as max size and allowed extensions.
                                                                                                                                                                                                                                                                      Expose the custom coach view boundary event onUploadRequested on the coach. �
                                                                                                                                                                                                                                                                      Then wire the flow:
                                                                                                                                                                                                                                                                      Main coach path -> stays on coach.
                                                                                                                                                                                                                                                                      Boundary event onUploadRequested -> to a server-side service or service flow call step -> then back to same coach or next step. IBM states that the boundary event must be wired to the next state, including loopback scenarios where the same coach reopens after data commit. �
                                                                                                                                                                                                                                                                      Calling the service flow
                                                                                                                                                                                                                                                                      After the boundary event, call SF_UploadFile.
                                                                                                                                                                                                                                                                      Input mapping
                                                                                                                                                                                                                                                                      Map:
                                                                                                                                                                                                                                                                      tw.local.uploadDataIn = tw.local.uploadData
                                                                                                                                                                                                                                                                      Output mapping
                                                                                                                                                                                                                                                                      Map result back:
                                                                                                                                                                                                                                                                      tw.local.uploadData = tw.local.uploadDataOut
                                                                                                                                                                                                                                                                      Inside the service flow, call your REST service and populate:
                                                                                                                                                                                                                                                                      uploadStatus
                                                                                                                                                                                                                                                                      documentId
                                                                                                                                                                                                                                                                      responseText
                                                                                                                                                                                                                                                                      errorMessage �
                                                                                                                                                                                                                                                                      Loop back to same coach
                                                                                                                                                                                                                                                                      After the service flow:
                                                                                                                                                                                                                                                                      Route back to the same coach if you want to show status on screen.
                                                                                                                                                                                                                                                                      Or route forward if upload success should continue the process. �
                                                                                                                                                                                                                                                                      IBM notes that in a loopback scenario only the affected data is reloaded instead of the full page, which is useful for showing the returned status in the same coach after upload. �
                                                                                                                                                                                                                                                                      Show returned status in coach
                                                                                                                                                                                                                                                                      Add this small update logic in the coach view load so the user sees the current server result after loopback:
                                                                                                                                                                                                                                                                      this.load = function() {
                                                                                                                                                                                                                                                                        var view = this;
                                                                                                                                                                                                                                                                          var root = this.context.element;
                                                                                                                                                                                                                                                                            var fileInput = root.find(".fu-file");
                                                                                                                                                                                                                                                                              var button = root.find(".fu-btn");
                                                                                                                                                                                                                                                                                var nameBox = root.find(".fu-name");
                                                                                                                                                                                                                                                                                  var statusBox = root.find(".fu-status");

                                                                                                                                                                                                                                                                                    var data = view.context.binding.get("value");
                                                                                                                                                                                                                                                                                      if (data) {
                                                                                                                                                                                                                                                                                          var fileName = data.get("fileName");
                                                                                                                                                                                                                                                                                              var uploadStatus = data.get("uploadStatus");
                                                                                                                                                                                                                                                                                                  var errorMessage = data.get("errorMessage");
                                                                                                                                                                                                                                                                                                      var documentId = data.get("documentId");

                                                                                                                                                                                                                                                                                                          if (fileName) {
                                                                                                                                                                                                                                                                                                                nameBox.text("Selected: " + fileName);
                                                                                                                                                                                                                                                                                                                    }

                                                                                                                                                                                                                                                                                                                        if (uploadStatus === "SUCCESS") {
                                                                                                                                                                                                                                                                                                                              statusBox.text("Upload successful. Document ID: " + documentId);
                                                                                                                                                                                                                                                                                                                                  } else if (uploadStatus === "ERROR") {
                                                                                                                                                                                                                                                                                                                                        statusBox.text("Upload failed. " + (errorMessage || ""));
                                                                                                                                                                                                                                                                                                                                            }
                                                                                                                                                                                                                                                                                                                                              }

                                                                                                                                                                                                                                                                                                                                                // keep the click handler code from previous example here
                                                                                                                                                                                                                                                                                                                                                };
                                                                                                                                                                                                                                                                                                                                                End-to-end mapping summary
                                                                                                                                                                                                                                                                                                                                                Important caveat
                                                                                                                                                                                                                                                                                                                                                If you ever call a service directly from a coach view and put complex BO output back into a binding, IBM warns that the returned object can contain @metadata, and that metadata may cause exceptions when later sent through a boundary event unless you remove it first. In your boundary-event design above, this problem is avoided because the coach view sends only your own clean upload payload fields. �
                                                                                                                                                                                                                                                                                                                                                What to build in Process Designer
                                                                                                                                                                                                                                                                                                                                                The exact items are:
                                                                                                                                                                                                                                                                                                                                                UploadData BO.
                                                                                                                                                                                                                                                                                                                                                SF_UploadFile service flow with input/output of type UploadData.
                                                                                                                                                                                                                                                                                                                                                Client-side human service with variable tw.local.uploadData.
                                                                                                                                                                                                                                                                                                                                                Coach containing CV_FileUpload.
                                                                                                                                                                                                                                                                                                                                                CV_FileUpload with boundary event onUploadRequested.
                                                                                                                                                                                                                                                                                                                                                Flow from onUploadRequested -> SF_UploadFile -> back to coach. �
                                                                                                                                                                                                                                                                                                                                                If you want, I can now give you the exact Process Designer tab-by-tab instructions with:
                                                                                                                                                                                                                                                                                                                                                Overview
                                                                                                                                                                                                                                                                                                                                                Variables
                                                                                                                                                                                                                                                                                                                                                HTML
                                                                                                                                                                                                                                                                                                                                                Behavior
                                                                                                                                                                                                                                                                                                                                                Coach binding
                                                                                                                                                                                                                                                                                                                                                Human service flow wiring
                                                                                                                                                                                                                                                                                                                                                Service flow input/output mapping