<p align="center">
  <h1 align="center">dstlled-diff</h1>
  <p align="center">
    <a href="https://github.com/dhth/dstlled-diff-action/releases/latest"><img alt="GitHub release" src="https://img.shields.io/github/release/dhth/dstlled-diff-action.svg?logo=github&style=flat-square"></a>
    <a href="https://github.com/marketplace/actions/dstlled-diff"><img alt="GitHub marketplace" src="https://img.shields.io/badge/marketplace-dstlled--diff--action-blue?logo=github&style=flat-square"></a>
    <a href="https://dhth.github.io/dstlled-diff-action"><img alt="GitHub marketplace" src="https://img.shields.io/website?url=https%3A%2F%2Fdhth.github.io%2Fdstlled-diff-action&style=flat-square&label=web-demo"></a>
  </p>
</p>

✨ Overview
---

`dstlled-diff` (short for "distilled-diff") is a GitHub action that processes a
specific git revision range and generates a diff that only includes changes in
the signatures of "code constructs" *(functions, methods, classes, traits,
interfaces, objects, type aliases, enums, etc.)*. It is powered by [dstll][1].

The main goal of `dstlled-diff` is to simplify the review of large structural
code changes by removing diff components that do not alter signatures.

![Usage](https://tools.dhruvs.space/images/dstlled-diff/dstlled-diff-1.png)

📜 Languages supported
---

- ![go](https://img.shields.io/badge/go-grey?logo=go)
- ![python](https://img.shields.io/badge/python-grey?logo=python)
- ![rust](https://img.shields.io/badge/rust-grey?logo=rust)
- ![scala 2](https://img.shields.io/badge/scala-grey?logo=scala)
- more to come

Δ Difference to regular git diff
---

👉 Consider this *(fairly long)* git diff:

<details><summary> expand </summary>

```diff
diff --git a/cmd/root.go b/cmd/root.go
index 9bdc5c8..8d40f9b 100644
--- a/cmd/root.go
+++ b/cmd/root.go
@@ -1,89 +1,94 @@
 package cmd
 
 import (
 	"context"
+	"errors"
 	"flag"
 	"fmt"
 	"os"
 	"strings"
 
 	"github.com/aws/aws-sdk-go-v2/config"
 	"github.com/aws/aws-sdk-go-v2/service/sqs"
 	"github.com/dhth/cueitup/ui"
 	"github.com/dhth/cueitup/ui/model"
 )
 
-func die(msg string, args ...any) {
-	fmt.Fprintf(os.Stderr, msg+"\n", args...)
-	os.Exit(1)
-}
+var (
+	errQueueURLEmpty        = errors.New("queue URL is empty")
+	errAWSProfileEmpty      = errors.New("AWS profile is empty")
+	errAWSRegionEmpty       = errors.New("AWS region is empty")
+	errQueueURLIncorrect    = errors.New("queue URL is incorrect")
+	errMsgFormatInvalid     = errors.New("message format is invalid")
+	errInvalidFlagUsage     = errors.New("invalid flag usage")
+	errCouldntLoadAWSConfig = errors.New("couldn't load AWS config")
+)
 
 var (
-	queueUrl   = flag.String("queue-url", "", "url of the queue to consume from")
+	queueURL   = flag.String("queue-url", "", "url of the queue to consume from")
 	awsProfile = flag.String("aws-profile", "", "aws profile to use")
 	awsRegion  = flag.String("aws-region", "", "aws region to use")
 	msgFormat  = flag.String("msg-format", "json", "message format")
 	subsetKey  = flag.String("subset-key", "", "extract a nested object inside the JSON body")
 	contextKey = flag.String("context-key", "", "the key to use as for context in the list")
 )
 
-func Execute() {
-
+func Execute() error {
 	flag.Usage = func() {
 		fmt.Fprintf(os.Stderr, "Inspect messages in an AWS SQS queue in a simple and deliberate manner.\n\nFlags:\n")
 		flag.PrintDefaults()
 		fmt.Fprintf(os.Stderr, "\n------\n%s", model.HelpText)
 	}
 	flag.Parse()
 
-	if *queueUrl == "" {
-		die("queue-url cannot be empty")
-	} else if !strings.HasPrefix(*queueUrl, "https://") {
-		die("queue-url must begin with https")
+	if *queueURL == "" {
+		return errQueueURLEmpty
+	}
+
+	if !strings.HasPrefix(*queueURL, "https://") {
+		return fmt.Errorf("%w: must begin with https", errQueueURLIncorrect)
 	}
 
 	if *awsProfile == "" {
-		die("aws-profile cannot be empty")
+		return errAWSProfileEmpty
 	}
 
 	if *awsRegion == "" {
-		die("aws-region cannot be empty")
+		return errAWSRegionEmpty
 	}
 
 	var msgFmt model.MsgFmt
 	switch *msgFormat {
 	case "json":
-		msgFmt = model.JsonFmt
+		msgFmt = model.JSONFmt
 	case "plaintext":
 		msgFmt = model.PlainTxtFmt
 	default:
-		die("cueitup only supports the following msg-format values: json, plaintext")
+		return fmt.Errorf("%w: supported values: json, plaintext", errMsgFormatInvalid)
 	}
 
-	if *subsetKey != "" && msgFmt != model.JsonFmt {
-		die("subset-key can only be used when msg-format=json")
+	if *subsetKey != "" && msgFmt != model.JSONFmt {
+		return fmt.Errorf("%w: subset-key can only be used when msg-format=json", errInvalidFlagUsage)
 	}
-	if *contextKey != "" && msgFmt != model.JsonFmt {
-		die("context-key can only be used when msg-format=json")
+	if *contextKey != "" && msgFmt != model.JSONFmt {
+		return fmt.Errorf("%w: context-key can only be used when msg-format=json", errInvalidFlagUsage)
 	}
 
 	msgConsumptionConf := model.MsgConsumptionConf{
 		Format:     msgFmt,
 		SubsetKey:  *subsetKey,
 		ContextKey: *contextKey,
 	}
 
 	sdkConfig, err := config.LoadDefaultConfig(context.TODO(),
 		config.WithSharedConfigProfile(*awsProfile),
 		config.WithRegion(*awsRegion),
 	)
 	if err != nil {
-		fmt.Println("Error:", err)
-		os.Exit(1)
+		return fmt.Errorf("%w: %s", errCouldntLoadAWSConfig, err.Error())
 	}
 
 	sqsClient := sqs.NewFromConfig(sdkConfig)
 
-	ui.RenderUI(sqsClient, *queueUrl, msgConsumptionConf)
-
+	return ui.RenderUI(sqsClient, *queueURL, msgConsumptionConf)
 }
diff --git a/cueitup.go b/main.go
similarity index 54%
rename from cueitup.go
rename to main.go
index 4a28df2..0480ca2 100644
--- a/cueitup.go
+++ b/main.go
@@ -1,9 +1,14 @@
 package main
 
 import (
+	"os"
+
 	"github.com/dhth/cueitup/cmd"
 )
 
 func main() {
-	cmd.Execute()
+	err := cmd.Execute()
+	if err != nil {
+		os.Exit(1)
+	}
 }
diff --git a/ui/model/cmds.go b/ui/model/cmds.go
index 837bbeb..038dbd3 100644
--- a/ui/model/cmds.go
+++ b/ui/model/cmds.go
@@ -8,91 +8,86 @@ import (
 	"strconv"
 	"strings"
 	"time"
 
 	"github.com/aws/aws-sdk-go-v2/aws"
 	"github.com/aws/aws-sdk-go-v2/service/sqs"
 	"github.com/aws/aws-sdk-go-v2/service/sqs/types"
 	tea "github.com/charmbracelet/bubbletea"
 )
 
-func (m model) FetchMessages(maxMessages int32, waitTime int32) tea.Cmd {
+func (m Model) FetchMessages(maxMessages int32, waitTime int32) tea.Cmd {
 	return func() tea.Msg {
-
 		var messages []types.Message
 		var messagesValues []string
 		var keyValues []string
 		result, err := m.sqsClient.ReceiveMessage(context.TODO(),
 			// WaitTimeSeconds > 0 enables long polling
 			// https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-short-and-long-polling.html#sqs-long-polling
 			&sqs.ReceiveMessageInput{
-				QueueUrl:            aws.String(m.queueUrl),
+				QueueUrl:            aws.String(m.queueURL),
 				MaxNumberOfMessages: maxMessages,
 				WaitTimeSeconds:     waitTime,
 				VisibilityTimeout:   30,
 			})
 		if err != nil {
 			return SQSMsgFetchedMsg{
 				messages: nil,
 				err:      err,
 			}
-		} else {
-			messages = result.Messages
-			for _, message := range messages {
-				msgValue, keyValue, _ := getMessageData(&message, m.msgConsumptionConf)
-				messagesValues = append(messagesValues, msgValue)
-				keyValues = append(keyValues, keyValue)
-			}
+		}
+		messages = result.Messages
+		for _, message := range messages {
+			msgValue, keyValue, _ := getMessageData(&message, m.msgConsumptionConf)
+			messagesValues = append(messagesValues, msgValue)
+			keyValues = append(keyValues, keyValue)
 		}
 
 		return SQSMsgFetchedMsg{
 			messages:      messages,
 			messageValues: messagesValues,
 			keyValues:     keyValues,
 			err:           nil,
 		}
 	}
 }
 
-func DeleteMessages(client *sqs.Client, queueUrl string, messages []types.Message) tea.Cmd {
+func DeleteMessages(client *sqs.Client, queueURL string, messages []types.Message) tea.Cmd {
 	return func() tea.Msg {
-
 		entries := make([]types.DeleteMessageBatchRequestEntry, len(messages))
 		for msgIndex := range messages {
 			entries[msgIndex].Id = aws.String(fmt.Sprintf("%v", msgIndex))
 			entries[msgIndex].ReceiptHandle = messages[msgIndex].ReceiptHandle
 		}
 		_, err := client.DeleteMessageBatch(context.TODO(),
 			&sqs.DeleteMessageBatchInput{
 				Entries:  entries,
-				QueueUrl: aws.String(queueUrl),
+				QueueUrl: aws.String(queueURL),
 			})
 		if err != nil {
 			return SQSMsgsDeletedMsg{
 				err: err,
 			}
 		}
 
 		return SQSMsgsDeletedMsg{}
 	}
 }
 
-func GetQueueMsgCount(client *sqs.Client, queueUrl string) tea.Cmd {
+func GetQueueMsgCount(client *sqs.Client, queueURL string) tea.Cmd {
 	return func() tea.Msg {
-
 		approxMsgCountType := types.QueueAttributeNameApproximateNumberOfMessages
 		attribute, err := client.GetQueueAttributes(context.TODO(),
 			&sqs.GetQueueAttributesInput{
-				QueueUrl:       aws.String(queueUrl),
+				QueueUrl:       aws.String(queueURL),
 				AttributeNames: []types.QueueAttributeName{approxMsgCountType},
 			})
-
 		if err != nil {
 			return QueueMsgCountFetchedMsg{
 				approxMsgCount: -1,
 				err:            err,
 			}
 		}
 
 		countStr := attribute.Attributes[string(approxMsgCountType)]
 		count, err := strconv.Atoi(countStr)
 		if err != nil {
@@ -103,32 +98,32 @@ func GetQueueMsgCount(client *sqs.Client, queueUrl string) tea.Cmd {
 		}
 		return QueueMsgCountFetchedMsg{
 			approxMsgCount: count,
 		}
 	}
 }
 
 func saveRecordValueToDisk(filePath string, msgValue string, msgFmt MsgFmt) tea.Cmd {
 	return func() tea.Msg {
 		dir := filepath.Dir(filePath)
-		err := os.MkdirAll(dir, 0755)
+		err := os.MkdirAll(dir, 0o755)
 		if err != nil {
 			return RecordSavedToDiskMsg{err: err}
 		}
 		var data string
 		switch msgFmt {
-		case JsonFmt:
+		case JSONFmt:
 			data = fmt.Sprintf("json\n%s\n", msgValue)
 		case PlainTxtFmt:
 			data = msgValue
 		}
-		err = os.WriteFile(filePath, []byte(data), 0644)
+		err = os.WriteFile(filePath, []byte(data), 0o644)
 		if err != nil {
 			return RecordSavedToDiskMsg{err: err}
 		}
 		return RecordSavedToDiskMsg{path: filePath}
 	}
 }
 
 func setContextSearchValues(userInput string) tea.Cmd {
 	return func() tea.Msg {
 		valuesEls := strings.Split(userInput, ",")
diff --git a/ui/model/delegate.go b/ui/model/delegate.go
index 80805d4..be199a1 100644
--- a/ui/model/delegate.go
+++ b/ui/model/delegate.go
@@ -11,35 +11,34 @@ func newAppItemDelegate() list.DefaultDelegate {
 	d := list.NewDefaultDelegate()
 
 	d.Styles.SelectedTitle = d.Styles.
 		SelectedTitle.
 		Foreground(lipgloss.Color(listColor)).
 		BorderLeftForeground(lipgloss.Color(listColor))
 	d.Styles.SelectedDesc = d.Styles.
 		SelectedTitle
 
 	d.UpdateFunc = func(msg tea.Msg, m *list.Model) tea.Cmd {
-		switch msgType := msg.(type) {
-		case tea.KeyMsg:
-			switch {
-			case key.Matches(msgType,
-				list.DefaultKeyMap().CursorUp,
-				list.DefaultKeyMap().CursorDown,
-				list.DefaultKeyMap().GoToStart,
-				list.DefaultKeyMap().GoToEnd,
-				list.DefaultKeyMap().NextPage,
-				list.DefaultKeyMap().PrevPage,
-			):
-				selected := m.SelectedItem()
-				if selected == nil {
-					return nil
-				}
-				key := selected.FilterValue()
-				return showItemDetails(key)
+		keyMsg, keyMsgOK := msg.(tea.KeyMsg)
+		if !keyMsgOK {
+			return nil
+		}
+		if key.Matches(keyMsg,
+			list.DefaultKeyMap().CursorUp,
+			list.DefaultKeyMap().CursorDown,
+			list.DefaultKeyMap().GoToStart,
+			list.DefaultKeyMap().GoToEnd,
+			list.DefaultKeyMap().NextPage,
+			list.DefaultKeyMap().PrevPage,
+		) {
+			selected := m.SelectedItem()
+			if selected == nil {
+				return nil
 			}
-
+			key := selected.FilterValue()
+			return showItemDetails(key)
 		}
 		return nil
 	}
 
 	return d
 }
diff --git a/ui/model/help.go b/ui/model/help.go
index 76c4bc5..7f9c7d6 100644
--- a/ui/model/help.go
+++ b/ui/model/help.go
@@ -1,61 +1,59 @@
 package model
 
 import "fmt"
 
-var (
-	HelpText = fmt.Sprintf(`
+var HelpText = fmt.Sprintf(`
   %s
 %s
   %s
 
   %s
 %s
   %s
 %s
   %s
 %s
 `,
-		helpHeaderStyle.Render("cueitup Reference Manual"),
-		helpSectionStyle.Render(`
+	helpHeaderStyle.Render("cueitup Reference Manual"),
+	helpSectionStyle.Render(`
   (scroll line by line with j/k/arrow keys or by half a page with <c-d>/<c-u>)
 
   cueitup has 3 views:
   - Message List View
   - Message Value View
   - Help View (this one)
 `),
-		helpHeaderStyle.Render("Keyboard Shortcuts"),
-		helpHeaderStyle.Render("General"),
-		helpSectionStyle.Render(`
+	helpHeaderStyle.Render("Keyboard Shortcuts"),
+	helpHeaderStyle.Render("General"),
+	helpSectionStyle.Render(`
       <tab>                          Switch focus to next section
       <s-tab>                        Switch focus to previous section
       1                              Maximize message value view
       ?                              Show help view
 `),
-		helpHeaderStyle.Render("Message List View"),
-		helpSectionStyle.Render(`
+	helpHeaderStyle.Render("Message List View"),
+	helpSectionStyle.Render(`
       h/<Up>                         Move cursor up
       k/<Down>                       Move cursor down
       n                              Fetch the next message from the queue
       N                              Fetch up to 10 more messages from the queue
       }                              Fetch up to 100 more messages from the queue
       d                              Toggle deletion mode; cueitup will delete messages
                                          after reading them
       <ctrl+s>                       Toggle contextual search prompt
       <ctrl+f>                       Toggle contextual filtering ON/OFF
       <ctrl+p>                       Toggle queue message count polling ON/OFF; ON by default
       p                              Toggle persist mode (cueitup will start persisting
                                          messages, at the location
                                          messages/<topic-name>/<timestamp-when-cueitup-started>/<unix-epoch>-<message-id>.md
       s                              Toggle skipping mode; cueitup will consume messages,
                                          but not populate its internal list, effectively
                                          skipping over them
 `),
-		helpHeaderStyle.Render("Message Value View   "),
-		helpSectionStyle.Render(`
+	helpHeaderStyle.Render("Message Value View   "),
+	helpSectionStyle.Render(`
       q                              Minimize section, and return focus to list view
       [,h                            Show details for the previous entry in the list
       ],l                            Show details for the next entry in the list
 `),
-	)
 )
diff --git a/ui/model/initial.go b/ui/model/initial.go
index e037600..28dd396 100644
--- a/ui/model/initial.go
+++ b/ui/model/initial.go
@@ -5,45 +5,44 @@ import (
 	"os"
 	"strings"
 	"time"
 
 	"github.com/aws/aws-sdk-go-v2/service/sqs"
 	"github.com/charmbracelet/bubbles/list"
 	"github.com/charmbracelet/bubbles/textinput"
 	"github.com/charmbracelet/lipgloss"
 )
 
-func InitialModel(sqsClient *sqs.Client, queueUrl string, msgConsumptionConf MsgConsumptionConf) model {
-
+func InitialModel(sqsClient *sqs.Client, queueURL string, msgConsumptionConf MsgConsumptionConf) Model {
 	appDelegate := newAppItemDelegate()
 	jobItems := make([]list.Item, 0)
 
-	queueParts := strings.Split(queueUrl, "/")
+	queueParts := strings.Split(queueURL, "/")
 	queueName := queueParts[len(queueParts)-1]
 	currentTime := time.Now()
 	timeString := currentTime.Format("2006-01-02-15-04-05")
 	persistDir := fmt.Sprintf("messages/%s/%s", queueName, timeString)
 
 	ti := textinput.New()
 	ti.Prompt = fmt.Sprintf("Filter messages where %s in > ", msgConsumptionConf.ContextKey)
 	ti.Focus()
 	ti.CharLimit = 100
 	ti.Width = 100
 
 	var dbg bool
 	if len(os.Getenv("DEBUG")) > 0 {
 		dbg = true
 	}
 
-	m := model{
+	m := Model{
 		sqsClient:            sqsClient,
-		queueUrl:             queueUrl,
+		queueURL:             queueURL,
 		msgConsumptionConf:   msgConsumptionConf,
 		pollForQueueMsgCount: true,
 		msgsList:             list.New(jobItems, appDelegate, listWidth+10, 0),
 		recordValueStore:     make(map[string]string),
 		persistDir:           persistDir,
 		contextSearchInput:   ti,
 		showHelpIndicator:    true,
 		debugMode:            dbg,
 		firstFetch:           true,
 	}
diff --git a/ui/model/model.go b/ui/model/model.go
index 73f2a00..cce4fc8 100644
--- a/ui/model/model.go
+++ b/ui/model/model.go
@@ -15,36 +15,36 @@ type stateView uint
 const (
 	msgsListView stateView = iota
 	msgValueView
 	helpView
 	contextualSearchView
 )
 
 type MsgFmt uint
 
 const (
-	JsonFmt MsgFmt = iota
+	JSONFmt MsgFmt = iota
 	PlainTxtFmt
 )
 
 const msgCountTickInterval = time.Second * 3
 
 type MsgConsumptionConf struct {
 	Format     MsgFmt
 	SubsetKey  string
 	ContextKey string
 }
 
-type model struct {
+type Model struct {
 	deserializationFmt   MsgFmt
 	sqsClient            *sqs.Client
-	queueUrl             string
+	queueURL             string
 	msgConsumptionConf   MsgConsumptionConf
 	activeView           stateView
 	lastView             stateView
 	pollForQueueMsgCount bool
 	msgsList             list.Model
 	helpVP               viewport.Model
 	showHelpIndicator    bool
 	msgValueVP           viewport.Model
 	recordValueStore     map[string]string
 	contextSearchInput   textinput.Model
@@ -58,17 +58,17 @@ type model struct {
 	helpVPReady          bool
 	vpFullScreen         bool
 	terminalWidth        int
 	terminalHeight       int
 	message              string
 	errorMsg             string
 	debugMode            bool
 	firstFetch           bool
 }
 
-func (m model) Init() tea.Cmd {
+func (m Model) Init() tea.Cmd {
 	return tea.Batch(
-		GetQueueMsgCount(m.sqsClient, m.queueUrl),
+		GetQueueMsgCount(m.sqsClient, m.queueURL),
 		tickEvery(msgCountTickInterval),
 		hideHelp(time.Minute*1),
 	)
 }
diff --git a/ui/model/msgs.go b/ui/model/msgs.go
index 15ec498..e1e52a4 100644
--- a/ui/model/msgs.go
+++ b/ui/model/msgs.go
@@ -1,18 +1,20 @@
 package model
 
 import (
 	"github.com/aws/aws-sdk-go-v2/service/sqs/types"
 )
 
-type MsgCountTickMsg struct{}
-type HideHelpMsg struct{}
+type (
+	MsgCountTickMsg struct{}
+	HideHelpMsg     struct{}
+)
 
 type SQSMsgFetchedMsg struct {
 	messages      []types.Message
 	messageValues []string
 	keyValues     []string
 	err           error
 }
 
 type QueueMsgCountFetchedMsg struct {
 	approxMsgCount int
diff --git a/ui/model/types.go b/ui/model/types.go
index 565b8cb..48c5646 100644
--- a/ui/model/types.go
+++ b/ui/model/types.go
@@ -18,12 +18,12 @@ func (item msgItem) Title() string {
 }
 
 func (item msgItem) Description() string {
 	if item.contextKeyValue != "" {
 		return RightPadTrim(fmt.Sprintf("%s: %s", RightPadTrim(item.contextKeyName, 10), item.contextKeyValue), listWidth)
 	}
 	return ""
 }
 
 func (item msgItem) FilterValue() string {
-	return string(*item.message.MessageId)
+	return *item.message.MessageId
 }
diff --git a/ui/model/update.go b/ui/model/update.go
index ba99b5b..41ae3b9 100644
--- a/ui/model/update.go
+++ b/ui/model/update.go
@@ -3,23 +3,26 @@ package model
 import (
 	"fmt"
 	"time"
 
 	"github.com/charmbracelet/bubbles/list"
 	"github.com/charmbracelet/bubbles/viewport"
 	tea "github.com/charmbracelet/bubbletea"
 	"github.com/tidwall/pretty"
 )
 
-const useHighPerformanceRenderer = false
+const (
+	useHighPerformanceRenderer = false
+	fetchingIndicator          = " ..."
+)
 
-func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
+func (m Model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
 	var cmds []tea.Cmd
 	m.message = ""
 	m.errorMsg = ""
 
 	switch msg := msg.(type) {
 	case tea.KeyMsg:
 		switch msg.String() {
 		case "ctrl+c", "q":
 			switch m.activeView {
 			case msgsListView:
@@ -41,31 +44,31 @@ func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
 		case "enter":
 			if m.activeView == contextualSearchView {
 				m.activeView = m.lastView
 				if len(m.contextSearchInput.Value()) > 0 {
 					cmds = append(cmds, setContextSearchValues(m.contextSearchInput.Value()))
 				} else {
 					m.filterMessages = false
 				}
 			}
 		case "n", " ":
-			m.message = " ..."
+			m.message = fetchingIndicator
 			cmds = append(cmds, m.FetchMessages(1, 0))
 		case "N":
-			m.message = " ..."
+			m.message = fetchingIndicator
 			for i := 0; i < 10; i++ {
 				cmds = append(cmds,
 					m.FetchMessages(1, 0),
 				)
 			}
 		case "}":
-			m.message = " ..."
+			m.message = fetchingIndicator
 			for i := 0; i < 20; i++ {
 				cmds = append(cmds,
 					m.FetchMessages(5, 0),
 				)
 			}
 		case "?":
 			m.lastView = m.activeView
 			m.activeView = helpView
 		case "d":
 			if m.activeView == msgsListView {
@@ -96,21 +99,21 @@ func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
 				selected := m.msgsList.SelectedItem()
 				if selected != nil {
 					result := string(pretty.Color([]byte(m.recordValueStore[selected.FilterValue()]), nil))
 					m.msgValueVP.SetContent(result)
 				}
 			}
 		case "ctrl+p":
 			m.pollForQueueMsgCount = !m.pollForQueueMsgCount
 			if m.pollForQueueMsgCount {
 				cmds = append(cmds,
-					tea.Batch(GetQueueMsgCount(m.sqsClient, m.queueUrl),
+					tea.Batch(GetQueueMsgCount(m.sqsClient, m.queueURL),
 						tickEvery(msgCountTickInterval),
 					),
 				)
 			}
 		case "ctrl+s":
 			if m.activeView == msgsListView {
 				m.lastView = m.activeView
 				m.activeView = contextualSearchView
 			}
 		case "ctrl+f":
@@ -184,82 +187,84 @@ func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
 		m.contextSearchInput.SetValue("")
 		m.filterMessages = true
 
 	case HideHelpMsg:
 		m.showHelpIndicator = false
 
 	case SQSMsgFetchedMsg:
 		if msg.err != nil {
 			m.errorMsg = msg.err.Error()
 		} else {
-			switch m.skipRecords {
-			case false:
+			if !m.skipRecords {
 				vPresenceMap := make(map[string]bool)
 				if m.filterMessages && len(m.contextSearchValues) > 0 {
 					for _, p := range m.contextSearchValues {
 						vPresenceMap[p] = true
 					}
 				}
 				for i, message := range msg.messages {
-
 					// only save/persist values that are requested to be filtered
 					if m.filterMessages && !(msg.keyValues[i] != "" && vPresenceMap[msg.keyValues[i]]) {
 						continue
 					}
 
 					m.msgsList.InsertItem(len(m.msgsList.Items()),
-						msgItem{message: message,
+						msgItem{
+							message:         message,
 							messageValue:    msg.messageValues[i],
 							contextKeyName:  m.msgConsumptionConf.ContextKey,
 							contextKeyValue: msg.keyValues[i],
 						},
 					)
 					m.recordValueStore[*message.MessageId] = msg.messageValues[i]
+
 					if m.persistRecords {
 						prefix := time.Now().Unix()
 						filePath := fmt.Sprintf("%s/%d-%s.md", m.persistDir, prefix, *message.MessageId)
 						cmds = append(cmds,
 							saveRecordValueToDisk(
 								filePath,
 								msg.messageValues[i],
 								m.msgConsumptionConf.Format,
 							),
 						)
 					}
 				}
+
 				if m.deleteMsgs {
 					cmds = append(cmds,
 						DeleteMessages(m.sqsClient,
-							m.queueUrl,
+							m.queueURL,
 							msg.messages),
 					)
 				}
+
 				if m.firstFetch {
 					selected := m.msgsList.SelectedItem()
 					if selected != nil {
 						result := string(pretty.Color([]byte(m.recordValueStore[selected.FilterValue()]), nil))
 						m.msgValueVP.SetContent(result)
 						m.firstFetch = false
 					}
 				}
 			}
 		}
 	case KMsgChosenMsg:
 		switch m.deserializationFmt {
-		case JsonFmt:
+		case JSONFmt:
 			result := string(pretty.Color([]byte(m.recordValueStore[msg.key]), nil))
 			m.msgValueVP.SetContent(result)
 		default:
 			m.msgValueVP.SetContent(m.recordValueStore[msg.key])
 		}
 	case MsgCountTickMsg:
-		cmds = append(cmds, GetQueueMsgCount(m.sqsClient, m.queueUrl))
+		cmds = append(cmds, GetQueueMsgCount(m.sqsClient, m.queueURL))
 		if m.pollForQueueMsgCount {
 			cmds = append(cmds, tickEvery(msgCountTickInterval))
 		}
 	case QueueMsgCountFetchedMsg:
 		if msg.err != nil {
 			m.errorMsg = msg.err.Error()
 		} else {
 			m.msgsList.Title = fmt.Sprintf("Messages (%d in queue)", msg.approxMsgCount)
 		}
 	}
diff --git a/ui/model/utils.go b/ui/model/utils.go
index 21bdc83..24bb3d3 100644
--- a/ui/model/utils.go
+++ b/ui/model/utils.go
@@ -80,30 +80,29 @@ func getRecordValueJSONNested(message *types.Message, extractKey string, context
 	}
 
 	return string(nestedBytes), contextualValue.(string), nil
 }
 
 func getMessageData(message *types.Message, msgConsumptionConf MsgConsumptionConf) (string, string, error) {
 	var msgValue, keyValue string
 	var err error
 
 	switch msgConsumptionConf.Format {
-	case JsonFmt:
+	case JSONFmt:
 		if msgConsumptionConf.SubsetKey != "" {
 			msgValue,
 				keyValue,
 				err = getRecordValueJSONNested(message,
 				msgConsumptionConf.SubsetKey,
 				msgConsumptionConf.ContextKey,
 			)
 		} else {
 			msgValue, err = getRecordValueJSONFull(message)
 		}
 	case PlainTxtFmt:
 		msgValue = *message.Body
 	}
 	if err != nil {
 		return "", "", err
-	} else {
-		return msgValue, keyValue, nil
 	}
+	return msgValue, keyValue, nil
 }
diff --git a/ui/model/view.go b/ui/model/view.go
index 5de1437..5e2bf63 100644
--- a/ui/model/view.go
+++ b/ui/model/view.go
@@ -1,23 +1,21 @@
 package model
 
 import (
 	"fmt"
 
 	"github.com/charmbracelet/lipgloss"
 )
 
-var (
-	listWidth = 50
-)
+var listWidth = 50
 
-func (m model) View() string {
+func (m Model) View() string {
 	var content string
 	var footer string
 	var mode string
 	var statusBar string
 	var debugMsg string
 	var msgValVPTitleStyle lipgloss.Style
 
 	if m.message != "" {
 		statusBar = Trim(m.message, 120)
 	}
diff --git a/ui/ui.go b/ui/ui.go
index f13c6e7..381c448 100644
--- a/ui/ui.go
+++ b/ui/ui.go
@@ -1,27 +1,26 @@
 package ui
 
 import (
+	"errors"
 	"fmt"
-	"log"
 	"os"
 
 	"github.com/aws/aws-sdk-go-v2/service/sqs"
 	tea "github.com/charmbracelet/bubbletea"
 	"github.com/dhth/cueitup/ui/model"
 )
 
-func RenderUI(sqsClient *sqs.Client, queueUrl string, msgConsumptionConf model.MsgConsumptionConf) {
+var errFailedToConfigureDebugging = errors.New("failed to configure debugging")
 
+func RenderUI(sqsClient *sqs.Client, queueURL string, msgConsumptionConf model.MsgConsumptionConf) error {
 	if len(os.Getenv("DEBUG")) > 0 {
 		f, err := tea.LogToFile("debug.log", "debug")
 		if err != nil {
-			fmt.Println("fatal:", err)
-			os.Exit(1)
+			return fmt.Errorf("%w: %s", errFailedToConfigureDebugging, err.Error())
 		}
 		defer f.Close()
 	}
-	p := tea.NewProgram(model.InitialModel(sqsClient, queueUrl, msgConsumptionConf), tea.WithAltScreen())
-	if _, err := p.Run(); err != nil {
-		log.Fatalf("Something went wrong %s", err)
-	}
+	p := tea.NewProgram(model.InitialModel(sqsClient, queueURL, msgConsumptionConf), tea.WithAltScreen())
+	_, err := p.Run()
+	return err
 }
```

</details>

👉 The "distilled-diff" version of the same diff looks like the following:

<details><summary> expand </summary>

```diff
diff --git a/5c52f68/cmd/root.go b/941d77f/cmd/root.go
index 7e245d5..a1c36f6 100644
--- a/5c52f68/cmd/root.go
+++ b/941d77f/cmd/root.go
@@ -1,2 +1 @@
-func die(msg string, args ...any)
-func Execute()
+func Execute() error
diff --git a/5c52f68/cueitup.go b/941d77f/main.go
similarity index 100%
rename from 5c52f68/cueitup.go
rename to 941d77f/main.go
diff --git a/5c52f68/ui/model/cmds.go b/941d77f/ui/model/cmds.go
index 35aaf4d..e429709 100644
--- a/5c52f68/ui/model/cmds.go
+++ b/941d77f/ui/model/cmds.go
@@ -1,2 +1,2 @@
-func DeleteMessages(client *sqs.Client, queueUrl string, messages []types.Message) tea.Cmd
-func GetQueueMsgCount(client *sqs.Client, queueUrl string) tea.Cmd
+func DeleteMessages(client *sqs.Client, queueURL string, messages []types.Message) tea.Cmd
+func GetQueueMsgCount(client *sqs.Client, queueURL string) tea.Cmd
@@ -8 +8 @@ func hideHelp(interval time.Duration) tea.Cmd
-func (m model) FetchMessages(maxMessages int32, waitTime int32) tea.Cmd
+func (m Model) FetchMessages(maxMessages int32, waitTime int32) tea.Cmd
diff --git a/5c52f68/ui/model/initial.go b/941d77f/ui/model/initial.go
index 1381c76..7654528 100644
--- a/5c52f68/ui/model/initial.go
+++ b/941d77f/ui/model/initial.go
@@ -1 +1 @@
-func InitialModel(sqsClient *sqs.Client, queueUrl string, msgConsumptionConf MsgConsumptionConf) model
+func InitialModel(sqsClient *sqs.Client, queueURL string, msgConsumptionConf MsgConsumptionConf) Model
diff --git a/5c52f68/ui/model/model.go b/941d77f/ui/model/model.go
index 30e48cb..992323e 100644
--- a/5c52f68/ui/model/model.go
+++ b/941d77f/ui/model/model.go
@@ -8 +8 @@ type MsgConsumptionConf struct {
-type model struct {
+type Model struct {
@@ -11 +11 @@ type model struct {
-	queueUrl             string
+	queueURL             string
@@ -38 +38 @@ type model struct {
-func (m model) Init() tea.Cmd
+func (m Model) Init() tea.Cmd
diff --git a/5c52f68/ui/model/msgs.go b/941d77f/ui/model/msgs.go
index 38d7c0c..48a036f 100644
--- a/5c52f68/ui/model/msgs.go
+++ b/941d77f/ui/model/msgs.go
@@ -1,2 +1,4 @@
-type MsgCountTickMsg struct{}
-type HideHelpMsg struct{}
+type (
+	MsgCountTickMsg struct{}
+	HideHelpMsg     struct{}
+)
diff --git a/5c52f68/ui/model/update.go b/941d77f/ui/model/update.go
index bf16682..3a8f3d0 100644
--- a/5c52f68/ui/model/update.go
+++ b/941d77f/ui/model/update.go
@@ -1 +1 @@
-func (m model) Update(tea.Model, tea.Cmd)
+func (m Model) Update(tea.Model, tea.Cmd)
diff --git a/5c52f68/ui/model/view.go b/941d77f/ui/model/view.go
index c62ab48..80c398e 100644
--- a/5c52f68/ui/model/view.go
+++ b/941d77f/ui/model/view.go
@@ -1 +1 @@
-func (m model) View() string
+func (m Model) View() string
diff --git a/5c52f68/ui/ui.go b/941d77f/ui/ui.go
index f8e0ecf..ebdc12a 100644
--- a/5c52f68/ui/ui.go
+++ b/941d77f/ui/ui.go
@@ -1 +1 @@
-func RenderUI(sqsClient *sqs.Client, queueUrl string, msgConsumptionConf model.MsgConsumptionConf)
+func RenderUI(sqsClient *sqs.Client, queueURL string, msgConsumptionConf model.MsgConsumptionConf) error
```

</details>

As seen above, the "distilled" version only shows changes in function
signatures, and type definitions, which makes maks reviewing large-scale
refactors easier.

⚡️ Usage
---

```yaml
name: dstlled-diff

on:
  pull_request:

jobs:
  dstlled-diff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - id: get-dstlled-diff
        uses: dhth/dstlled-diff-action@v0.1.0
        with:
          starting-commit: ${{ github.event.pull_request.base.sha }}
          ending-commit: ${{ github.event.pull_request.head.sha }}
          post-comment-on-pr: 'true'
```

The "distilled" diff will be available as an output of the action, via
`steps.get-dstlled-diff.outputs.diff`.

🔡 Inputs
---

Following inputs can be used as `step.with` keys:

| Name                 | Type   | Default | Description                                                        |
|----------------------|--------|---------|--------------------------------------------------------------------|
| `starting-commit`    | String |         | Starting commit for the git revision range                         |
| `ending-commit`      | String |         | Ending commit for the git revision range                           |
| `pattern`            | String | `*`     | Pattern to run dstlled-diff on                                     |
| `directory`          | String | `.`     | Working directory (below repository root)                          |
| `post-comment-on-pr` | Bool   | `false` | Post comment containing dstlled-diff to corresponding pull request |
| `save-diff-to-file`  | Bool   | `false` | Save diff to a local file called `diff.patch`                      |

⚙️ Other use cases
---

### Web Interface

The output of `dstlled-diff` can be rendered in a web view (see it running
[here][2]), as seen in the image below. Code for this can be found
[here](./.github/workflows/web-demo.yml).

![web-demo](https://tools.dhruvs.space/images/dstlled-diff/dstlled-diff-2.png)

[1]: https://github.com/dhth/dstll
[2]: https://dhth.github.io/dstlled-diff-action/
