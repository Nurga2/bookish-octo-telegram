require "import"
import "android.widget.*"
import "android.view.View"
import "android.content.Context"
import "android.text.InputType"
import "cjson"
import "android.content.Intent"
import "android.net.Uri"
import "android.media.MediaPlayer"
import "android.widget.VideoView"
import "android.widget.MediaController"
import "java.util.HashMap"
import "java.net.URLEncoder"
import "android.app.DownloadManager"
import "android.os.Environment"
import "android.graphics.Typeface"

activity = this

package.loaded["config"] = nil
local config = require("config")

local videoOptions = {}
local selectedUrl = nil
local trackTitle = "Media_Download"

function vibrate()
	local vibrator = activity.getSystemService(Context.VIBRATOR_SERVICE)
	if vibrator then vibrator.vibrate(35) end
	end

	function urlEncode(str)
		if str then return URLEncoder.encode(str, "UTF-8") end
			return ""
		end

		function cleanName(str)
			if not str then return "Media_" .. os.time() end
				local s = str:gsub("[^a-zA-Z0-9%s%-_]", "")
				return s:sub(1, 50)
			end

			function startDownload(url, title, subname)
				pcall(function()
					local dm = activity.getSystemService(Context.DOWNLOAD_SERVICE)
					local request = DownloadManager.Request(Uri.parse(url))
					request.addRequestHeader("User-Agent", "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36")
					request.setNotificationVisibility(1)

					local fileName = cleanName(title) .. "_" .. subname .. ".mp4"
					request.setTitle(title .. " (" .. subname .. ")")
					request.setDescription("Downloading...")
					request.setDestinationInExternalPublicDir(Environment.DIRECTORY_DOWNLOADS, "MyPluginDownloads/" .. fileName)

					dm.enqueue(request)
					service.speak("Download started")
				end)
		end

		local item_layout = {
			LinearLayout,
			orientation="vertical",
			layout_width="fill",
			layout_height="wrap_content",
			padding="8dp",
			{
				LinearLayout,
				orientation="vertical",
				layout_width="fill",
				layout_height="wrap_content",
				backgroundColor=0xFF333333,
				padding="15dp",
				{
					TextView,
					id="opt_res",
					textSize="18sp",
					typeface=Typeface.DEFAULT_BOLD,
					textColor=0xFFFFFFFF,
					},
				{
					TextView,
					id="opt_size",
					textSize="14sp",
					textColor=0xFFCCCCCC,
					layout_marginTop="5dp"
					}
				}
			}

		layout = {
			LinearLayout,
			orientation = "vertical",
			layout_width = "fill",
			layout_height = "fill",
			padding = "10dp",
			{
				EditText,
				id = "textInput",
				hint = "Paste Link Here...",
				layout_width = "fill",
				layout_height = "wrap_content",
				padding = "10dp",
				lines = 1,
				inputType = InputType.TYPE_CLASS_TEXT,
				},
			{
				Button,
				id = "scanButton",
				text = "Process Link",
				layout_width = "fill",
				layout_height = "wrap_content",
				layout_marginTop = "5dp"
				},

			{
				FrameLayout,
				layout_width = "fill",
				layout_height = "200dp",
				layout_marginTop = "10dp",
				backgroundColor = "0xFF000000",
				{
					VideoView,
					id = "videoDisplay",
					layout_width = "match",
					layout_height = "match",
					layout_gravity = "center",
					visibility = View.GONE
					},
				{
					TextView,
					id = "statusText",
					text = "Paste link and click Process.",
					textColor = "0xFF888888",
					layout_gravity = "center",
					visibility = View.VISIBLE,
					gravity = "center",
					padding="10dp"
					}
				},

			{
				TextView,
				id="listLabel",
				text = "Multiple options found:",
				textColor = 0xFFCCCCCC,
				layout_marginTop = "10dp",
				paddingLeft = "5dp",
				visibility = "gone"
				},

			{
				ListView,
				id = "optionsList",
				layout_width = "fill",
				layout_height = "0dp",
				layout_weight = 1,
				dividerHeight = "0",
				visibility = "gone"
				},

			{
				LinearLayout,
				id="finalInfoLayout",
				orientation="vertical",
				layout_width="fill",
				layout_height="0dp",
				layout_weight=1,
				padding="10dp",
				visibility="gone",
				gravity="center",
				{
					TextView,
					text="✅ Direct Link Found",
					textSize="20sp",
					textColor=0xFF00FF00,
					typeface=Typeface.DEFAULT_BOLD,
					gravity="center"
					},
				{
					TextView,
					id="finalTitleText",
					text="Media Ready",
					textSize="16sp",
					textColor=0xFFFFFFFF,
					gravity="center",
					layout_marginTop="10dp"
					},
				{
					TextView,
					id="finalDetailText",
					text="Tap Play or Download",
					textSize="14sp",
					textColor=0xFFAAAAAA,
					gravity="center",
					layout_marginTop="5dp"
					},

				{
					Button,
					id="backToListBtn",
					text = "Back to List",
					layout_marginTop="20dp",
					visibility="gone",
					onClick=function()
					finalInfoLayout.setVisibility(View.GONE)
					optionsList.setVisibility(View.VISIBLE)
					listLabel.setVisibility(View.VISIBLE)
					playButton.setEnabled(false)
					downloadButton.setEnabled(false)
				end
				}
			},

		{
			LinearLayout,
			orientation = "horizontal",
			layout_width = "fill",
			layout_height = "wrap_content",
			layout_marginTop = "5dp",
			{
				Button,
				id = "playButton",
				text = "Play",
				layout_width = "0dp",
				layout_weight = 1,
				enabled = false
				},
			{
				Button,
				id = "downloadButton",
				text = "Download",
				layout_width = "0dp",
				layout_weight = 1,
				enabled = false
				},
			},
		{
			Button,
			text = "Close",
			layout_width = "match",
			layout_height = "wrap_content",
			onClick = function()
			videoDisplay.stopPlayback()
			dlg.dismiss()
		end
		}
	}

dlg = LuaDialog(this)
dlg.setTitle("Platforms supported for downloading TikTok, YouTube, Spotify, Instagram, and Facebook")
dlg.setView(loadlayout(layout))
dlg.setCancelable(false)

function updateStatus(txt)
	statusText.text = txt
	statusText.setVisibility(View.VISIBLE)
	videoDisplay.setVisibility(View.GONE)
	service.speak(txt)
end

function resetUI()
	videoOptions = {}
	playButton.setEnabled(false)
	downloadButton.setEnabled(false)
	videoDisplay.stopPlayback()
	optionsList.setAdapter(nil)

	optionsList.setVisibility(View.GONE)
	listLabel.setVisibility(View.GONE)
	finalInfoLayout.setVisibility(View.GONE)
	backToListBtn.setVisibility(View.GONE)
end

function showFinalPanel(url, title, fromList)
	selectedUrl = url

	finalTitleText.text = title or "Media Found"
	finalDetailText.text = "Ready to Stream/Save"

	optionsList.setVisibility(View.GONE)
	listLabel.setVisibility(View.GONE)
	finalInfoLayout.setVisibility(View.VISIBLE)

	if fromList then
		backToListBtn.setVisibility(View.VISIBLE)
	else
		backToListBtn.setVisibility(View.GONE)
	end

	playButton.setEnabled(true)
	downloadButton.setEnabled(true)
	updateStatus("Media Ready.")
end

scanButton.onClick = function()
vibrate()
local input_url = textInput.text
if #input_url == 0 then updateStatus("Paste URL first.") return end

	resetUI()
	updateStatus("Processing...")

	local target_url = string.format("%s?url=%s", config.Social_Downloader_api, urlEncode(input_url))

	Thread(Runnable {
			run = function()
			Http.get(target_url, function(code, content)
				if code ~= 200 then updateStatus("HTTP Error: " .. code) return end

					local status, jsonData = pcall(cjson.decode, content)
					if not status then updateStatus("JSON Parse Error") return end

						if jsonData.status == true and jsonData.result then
							local res = jsonData.result
							trackTitle = "Media_" .. (jsonData.creator or "Hazel")

							local links = res.download_links

							if links and #links > 0 then

								if #links == 1 then

									showFinalPanel(links[1], "Direct Link", false)
									service.speak("Direct link found.")

								else

									for i, link in ipairs(links) do
										table.insert(videoOptions, {
												name = "Option " .. i,
												url = link
												})
									end

									updateStatus("Found " .. #links .. " options.")

									local adapterData = {}
									for k, v in ipairs(videoOptions) do
										table.insert(adapterData, {
												opt_res = v.name,
												opt_size = "Tap to Select"
												})
									end

									local adapter = LuaAdapter(activity, adapterData, item_layout)
									optionsList.setAdapter(adapter)

									optionsList.setVisibility(View.VISIBLE)
									listLabel.setVisibility(View.VISIBLE)
								end

							else
								updateStatus("No download links found.")
							end
						else
							updateStatus("API Failed: " .. tostring(jsonData.message))
						end
					end)
			end
			}).start()
end

optionsList.onItemClick = function(parent, view, position, id)
vibrate()
local selected = videoOptions[position + 1]
if selected then
	showFinalPanel(selected.url, selected.name, true)
end
end

playButton.onClick = function()
vibrate()
if not selectedUrl then return end

	if videoDisplay.isPlaying() then
		videoDisplay.pause()
		playButton.text = "Resume"
	else
		statusText.setVisibility(View.GONE)
		videoDisplay.setVisibility(View.VISIBLE)

		local mc = MediaController(activity)
		videoDisplay.setMediaController(mc)

		videoDisplay.setVideoURI(Uri.parse(selectedUrl))
		videoDisplay.setOnPreparedListener(MediaPlayer.OnPreparedListener{
				onPrepared = function(mp)
				videoDisplay.start()
				playButton.text = "Pause"
				service.speak("Playing.")
			end
			})
end
end

downloadButton.onClick = function()
vibrate()
if selectedUrl then
	startDownload(selectedUrl, trackTitle, "Media")
end
end

service.playSoundTick()
dlg.show()
