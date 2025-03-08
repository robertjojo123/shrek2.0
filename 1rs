local monitor = peripheral.find("monitor")

if not monitor then
    error("No monitor found! Attach a monitor to use this script.", 0)
end

monitor.setTextScale(0.5)  -- Adjust text scale for large monitors

-- Set base video URL
local baseURL = "https://raw.githubusercontent.com/robertjojo123/shrek2.0/refs/heads/main/video_part_"

-- Get computer label and determine quadrant
local quadrant = os.getComputerLabel()
local quadrantIndex = tonumber(string.sub(quadrant, -1))  -- Extracts 0,1,2,3

if quadrantIndex == nil or quadrantIndex < 0 or quadrantIndex > 3 then
    error("Error: This computer's label must be 'comp0', 'comp1', 'comp2', or 'comp3'.", 0)
end

local firstVideoDuration = 38000  -- 38s
local otherVideoDuration = 45000  -- 45s
local frameInterval = 200          -- 200ms per frame
local linesPerFrame = 40           -- Each frame consists of 40 lines

function getMovieURL(index)
    return baseURL .. index .. "_q" .. quadrantIndex .. ".nfv"
end

function downloadVideo(index, filename)
    local url = getMovieURL(index)
    print("Downloading:", url)

    local response = http.get(url)
    if response then
        local file = fs.open(filename, "wb")
        file.write(response.readAll())
        file.close()
        response.close()
        return true
    else
        print("Failed to download:", filename)
    end
    return false
end

function loadVideo(videoFile)
    local videoData = {}
    for line in io.lines(videoFile) do
        table.insert(videoData, line)
    end
    local resolution = { videoData[1]:match("(%d+) (%d+)") }
    table.remove(videoData, 1)
    return videoData, resolution
end

function playVideo(videoFile, videoStartTime, videoIndex)
    local videoData, resolution = loadVideo(videoFile)
    if not videoData or not resolution then
        print("Error loading video data.")
        return
    end

    local frameWidth, frameHeight = tonumber(resolution[1]), tonumber(resolution[2])
    if not frameWidth or not frameHeight then
        print("Invalid resolution in video file.")
        return
    end

    local frameIndex = 1
    local videoEndTime = videoStartTime + (videoIndex == 1 and firstVideoDuration or otherVideoDuration)
    local frameStartTime = os.epoch("utc")

    -- Get monitor size
    local monitorWidth, monitorHeight = monitor.getSize()

    -- **Previous frame storage for flicker reduction**
    local previousFrame = {}

    function nextFrame()
        local currentTime = os.epoch("utc")
        local elapsedTime = currentTime - videoStartTime
        local expectedFrame = math.floor(elapsedTime / frameInterval) * frameHeight

        if expectedFrame > frameIndex then
            frameIndex = expectedFrame
        end

        -- **Extract full frame data from video file**
        local frameLines = {}
        for i = 1, frameHeight do
            if frameIndex + i > #videoData then
                break
            end
            table.insert(frameLines, videoData[frameIndex + i])
        end

        -- **Convert frame data into an image**
        if #frameLines > 0 then
            local imageData = paintutils.parseImage(table.concat(frameLines, "\n"))

            -- **Redirect output to the MONITOR**
            term.redirect(monitor)

            -- **Scale image manually (2x size)**
            local scaledImage = {}
            for y = 1, #imageData do
                local row = imageData[y]
                local newRow = {}
                for x = 1, #row do
                    -- **Duplicate each pixel horizontally (2x width)**
                    table.insert(newRow, row[x])
                    table.insert(newRow, row[x])
                end
                -- **Duplicate each row vertically (2x height)**
                table.insert(scaledImage, newRow)
                table.insert(scaledImage, newRow)
            end

            -- **Flicker Prevention: Only update pixels that changed**
            for y = 1, #scaledImage do
                for x = 1, #scaledImage[y] do
                    if not previousFrame[y] or previousFrame[y][x] ~= scaledImage[y][x] then
                        paintutils.drawPixel(x, y, scaledImage[y][x])
                    end
                end
            end

            -- **Store previous frame**
            previousFrame = scaledImage

            -- **Reset back to terminal after drawing**
            term.redirect(term.native())
        end

        frameIndex = frameIndex + frameHeight

        -- Stop playing if past the last frame
        if frameIndex > #videoData then
            return false
        end

        -- **Frame Timing (to catch up if needed)**
        local elapsedFrameTime = os.epoch("utc") - frameStartTime
        frameStartTime = os.epoch("utc")
        local sleepTime = (frameInterval - elapsedFrameTime) / 1000
        if sleepTime < 0 then
            sleepTime = 0
        end

        os.sleep(sleepTime)
        return true
    end

    while os.epoch("utc") < videoEndTime do
        if not nextFrame() then
            break
        end
    end
end

function playMovie()
    local videoIndex = 1
    local videoStartTime = os.epoch("utc")

    monitor.setBackgroundColor(colors.black)
    monitor.clear()
    
    print("Preparing to play movie...")
    os.sleep(0.75)  -- **750ms delay before first video starts**

    parallel.waitForAny(
        function()
            while true do
                print("Downloading video part:", videoIndex)
                if not downloadVideo(videoIndex, "/current_video.nfv") then 
                    print("No more video parts available, stopping playback.")
                    break 
                end

                local nextIndex = videoIndex + 1
                local nextFile = "/next_video.nfv"

                print("Pre-downloading next part:", nextIndex)
                downloadVideo(nextIndex, nextFile)

                print("Playing video part:", videoIndex)
                playVideo("/current_video.nfv", videoStartTime, videoIndex)
                fs.delete("/current_video.nfv")

                if fs.exists(nextFile) then
                    fs.move(nextFile, "/current_video.nfv")
                    videoIndex = nextIndex
                    videoStartTime = os.epoch("utc") + firstVideoDuration + ((videoIndex - 2) * otherVideoDuration)
                else
                    break
                end
            end
        end
    )
end

print("Waiting for redstone signal...")

while true do
    -- **Check for redstone signal from ANY side**
    if redstone.getInput("top") or redstone.getInput("bottom") or redstone.getInput("left") or redstone.getInput("right") or redstone.getInput("front") or redstone.getInput("back") then
        print("Redstone signal detected! Starting movie playback...")
        playMovie()
    end
    os.sleep(0.1)
end
