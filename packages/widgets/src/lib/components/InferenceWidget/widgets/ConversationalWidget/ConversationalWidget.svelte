<script lang="ts">
	import type { WidgetProps, ExampleRunOpts, InferenceRunOpts } from "../../shared/types.js";
	import type { WidgetExampleTextInput } from "@huggingface/tasks";

	import WidgetOutputConvo from "../../shared/WidgetOutputConvo/WidgetOutputConvo.svelte";
	import WidgetQuickInput from "../../shared/WidgetQuickInput/WidgetQuickInput.svelte";
	import WidgetWrapper from "../../shared/WidgetWrapper/WidgetWrapper.svelte";
	import { addInferenceParameters, callInferenceApi, updateUrl } from "../../shared/helpers.js";
	import { isTextInput } from "../../shared/inputValidation.js";
	import { widgetStates } from "../../stores.js";

	export let apiToken: WidgetProps["apiToken"];
	export let apiUrl: WidgetProps["apiUrl"];
	export let callApiOnMount: WidgetProps["callApiOnMount"];
	export let model: WidgetProps["model"];
	export let noTitle: WidgetProps["noTitle"];
	export let shouldUpdateUrl: WidgetProps["shouldUpdateUrl"];
	export let includeCredentials: WidgetProps["includeCredentials"];

	$: isDisabled = $widgetStates?.[model.id]?.isDisabled;

	interface Conversation {
		generated_responses: string[];
		past_user_inputs: string[];
	}
	interface Response {
		conversation: Conversation;
		generated_text: string;
	}

	type Output = Array<{
		input: string;
		response: string;
	}>;

	let computeTime = "";
	let conversation: {
		generated_responses: string[];
		past_user_inputs: string[];
	} = {
		generated_responses: [],
		past_user_inputs: [],
	};
	let error: string = "";
	let isLoading = false;
	let modelLoading = {
		isLoading: false,
		estimatedTime: 0,
	};
	let output: Output = [];
	let outputJson: string;
	let text = "";

	async function getOutput({
		withModelLoading = false,
		isOnLoadCall = false,
		exampleOutput = undefined,
	}: InferenceRunOpts = {}) {
		const trimmedText = text.trim();

		if (!trimmedText) {
			return;
		}

		if (shouldUpdateUrl && !conversation.past_user_inputs.length) {
			updateUrl({ text: trimmedText });
		}

		const requestBody = {
			inputs: {
				generated_responses: conversation.generated_responses,
				past_user_inputs: conversation.past_user_inputs,
				text: trimmedText,
			},
		};
		addInferenceParameters(requestBody, model);

		isLoading = true;

		const res = await callInferenceApi(
			apiUrl,
			model.id,
			requestBody,
			apiToken,
			parseOutput,
			withModelLoading,
			includeCredentials,
			isOnLoadCall
		);

		isLoading = false;
		// Reset values
		computeTime = "";
		error = "";
		modelLoading = { isLoading: false, estimatedTime: 0 };
		outputJson = "";

		if (res.status === "success") {
			computeTime = res.computeTime;
			outputJson = res.outputJson;
			if (res.output) {
				conversation = res.output.conversation;
				output = res.output.output;
			}
			// Emptying input value
			text = "";
		} else if (res.status === "loading-model") {
			modelLoading = {
				isLoading: true,
				estimatedTime: res.estimatedTime,
			};
			getOutput({ withModelLoading: true });
		} else if (res.status === "error") {
			error = res.error;
		}
	}

	function isValidOutput(arg: any): arg is Response {
		return (
			arg && Array.isArray(arg?.conversation?.generated_responses) && Array.isArray(arg?.conversation?.past_user_inputs)
		);
	}

	function parseOutput(body: unknown): {
		conversation: Conversation;
		output: Output;
	} {
		if (isValidOutput(body)) {
			const conversation = body.conversation;
			const pastUserInputs = conversation.past_user_inputs;
			const generatedResponses = conversation.generated_responses;
			const output = pastUserInputs
				.filter(
					(x, i) =>
						x !== null && x !== undefined && generatedResponses[i] !== null && generatedResponses[i] !== undefined
				)
				.map((x, i) => ({
					input: x ?? "",
					response: generatedResponses[i] ?? "",
				}));
			return { conversation, output };
		}
		throw new TypeError(
			"Invalid output: output must be of type <conversation: <generated_responses:Array; past_user_inputs:Array>>"
		);
	}

	function applyWidgetExample(sample: WidgetExampleTextInput, opts: ExampleRunOpts = {}) {
		text = sample.text;
		if (opts.isPreview) {
			return;
		}
		const exampleOutput = sample.output;
		getOutput({ ...opts.inferenceOpts, exampleOutput });
	}
</script>

<WidgetWrapper {apiUrl} {includeCredentials} {model} let:WidgetInfo let:WidgetHeader let:WidgetFooter>
	<WidgetHeader
		{noTitle}
		{model}
		{isLoading}
		{isDisabled}
		{callApiOnMount}
		{applyWidgetExample}
		validateExample={isTextInput}
	/>
	<WidgetOutputConvo modelId={model.id} {output} />

	<WidgetQuickInput
		bind:value={text}
		flatTop={true}
		{isLoading}
		{isDisabled}
		onClickSubmitBtn={() => {
			getOutput();
		}}
		submitButtonLabel="Send"
	/>

	<WidgetInfo {model} {computeTime} {error} {modelLoading} />

	<WidgetFooter {model} {isDisabled} {outputJson} />
</WidgetWrapper>
