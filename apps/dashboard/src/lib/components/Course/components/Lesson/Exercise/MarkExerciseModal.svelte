<script lang="ts">
  import Modal from '$lib/components/Modal/index.svelte';
  import PrimaryButton from '$lib/components/PrimaryButton/index.svelte';
  import { VARIANTS } from '$lib/components/PrimaryButton/constants';
  import Preview from './Preview.svelte';
  import { STATUS } from './constants';
  import { snackbar } from '$lib/components/Snackbar/store';
  import TextArea from '$lib/components/Form/TextArea.svelte';
  import { Dropdown, Tag } from 'carbon-components-svelte';
  import { useCompletion } from 'ai/svelte';
  import { QUESTION_TYPE } from '$lib/components/Question/constants';
  import { t } from '$lib/utils/functions/translations';
  import { OverflowMenu, OverflowMenuItem } from 'carbon-components-svelte';
  import type { SubmissionIdData } from '$lib/utils/types/submission';

  export let open = false;
  export let onClose = () => {};
  export let handleSave = (d: SubmissionIdData) => {};
  export let isGradeWithAI = false;
  export let data: SubmissionIdData;
  export let deleteSubmission = async (id: string, statusId: number) => {};
  export let updateStatus = (arg: {
    submissionId: string;
    prevStatusId: number;
    nextStatusId: number;
    total: number;
  }) => {};
  export let isSaving = false;

  const SELECTABLE_STATUS = [
    {
      id: STATUS.SUBMITTED,
      text: $t('course.navItem.submissions.submission_status.submitted')
    },
    {
      id: STATUS.IN_PROGRESS,
      text: $t('course.navItem.submissions.submission_status.in_progress')
    },
    {
      id: STATUS.GRADED,
      text: $t('course.navItem.submissions.submission_status.graded')
    }
  ];

  let status = SELECTABLE_STATUS[0];
  let selectedId = status.id;
  let reasons = {};
  let isLoading = false;
  let total = 0;
  let maxPoints = 0;
  let openMenu = false;
  let openDeletePrompt = false;
  let isDeleting = false;

  function getMaxPoints(questions) {
    return (questions || []).reduce((acc, question) => acc + question.points, 0);
  }

  function calculateTotal(grades: Record<string, string>): number {
    if (!grades) return 0;
    return Object.values(grades).reduce((acc, grade) => acc + parseInt(grade || '0'), 0);
  }

  function handleStatusChange(event) {
    const newSelectedId = event.detail.selectedId;

    setStatus({
      ...data,
      statusId: newSelectedId
    });

    updateStatus({
      submissionId: data.id,
      prevStatusId: data.statusId,
      nextStatusId: status.id,
      total
    });

    snackbar.success(`${$t('snackbar.exercise.submission_updated')} '${status.text}'`);
  }

  function setStatus(data: SubmissionIdData) {
    const statusBySelectedId = SELECTABLE_STATUS.find((status) => status.id === data.statusId);

    if (!statusBySelectedId) {
      return;
    }

    status = statusBySelectedId;

    if (data.statusId !== selectedId) {
      selectedId = data.statusId;
    }
  }

  async function handleDeleteSubmission() {
    isDeleting = true;
    await deleteSubmission(data.id, status.id);
    isDeleting = false;
  }

  const { input, handleSubmit, completion } = useCompletion({
    api: '/api/completion/gradingprompt',
    onFinish: async () => {
      try {
        const responseData = $completion.replace('```json', '').replace('```', '');

        let aiResponses: {
          id: number;
          score: number;
          explanation: string;
        }[] = [];
        try {
          // Parse the modified response data as JSON
          aiResponses = JSON.parse(responseData);
        } catch (error) {
          console.error('Error parsing AI response', error);
        }

        data?.questions.forEach((question) => {
          const { id, points, question_type_id } = question;

          if (question_type_id !== QUESTION_TYPE.TEXTAREA) {
            const answer = data.questionAnswers.find((q) => q.question_id === id);
            const answerPoints = answer?.answers?.length ?? 0;
            reasons = {
              ...reasons,
              [id]: `${$t(
                'course.navItem.submissions.grading_modal.questions_tried'
              )} ${answerPoints} `
            };
            data.questionAnswerByPoint[id] = `${Math.ceil(points / answerPoints)}`;
          } else if (aiResponses.length) {
            const graded = aiResponses.find((res) => res.id === id);

            reasons = {
              ...reasons,
              [id]: `${graded?.explanation}`
            };

            data.questionAnswerByPoint[id] = `${graded?.score}`;
          }
        });
      } catch (error) {
        console.error('Error', error);
      } finally {
        isLoading = false;
      }
    }
  });

  function gradeWithAI() {
    isGradeWithAI = true;
    isLoading = true;
    const paragraphAiInput = data.questions
      .filter((q) => q.question_type_id === QUESTION_TYPE.TEXTAREA)
      .map((q) => {
        const answer = data.questionAnswers.find((qA) => qA.question_id === q.id); // { open_answer: '' }
        return {
          id: q.id,
          question: q.title,
          answer: answer?.open_answer,
          point: q.points
        };
      });

    $input = JSON.stringify(paragraphAiInput);
    handleSubmit({ preventDefault: () => {} });
  }

  $: total = calculateTotal(data.questionAnswerByPoint);
  $: maxPoints = getMaxPoints(data.questions);
  $: setStatus(data);
</script>

<Modal
  modalHeading={data.title}
  bind:open
  {onClose}
  width="w-4/5 h-[90%]"
  containerClass="flex items-start !max-h-full h-[85%] py-0 px-4"
  headerClass="py-2"
  labelClass="text-base font-semibold"
>
  <div class="w-full h-full mt-2">
    {#if openDeletePrompt}
      <div class="w-96 mx-auto p-3 border border-gray-300 rounded-md">
        <h1 class="dark:text-white text-lg">
          {$t('delete_modal.content')}
        </h1>

        <div class="mt-5 flex items-center justify-between">
          <PrimaryButton
            className="px-6 py-3"
            variant={VARIANTS.OUTLINED}
            label={$t('delete_modal.no')}
            onClick={() => (openDeletePrompt = false)}
            isDisabled={isDeleting}
          />
          <PrimaryButton
            className="px-6 py-3"
            variant={VARIANTS.OUTLINED}
            label={$t('delete_modal.yes')}
            onClick={handleDeleteSubmission}
            isLoading={isDeleting}
          />
        </div>
      </div>
    {:else}
      <Preview
        questions={Array.isArray(data.questions)
          ? data.questions.sort((a, b) => a.order - b.order)
          : []}
        questionnaireMetaData={{
          answers: data.answers || {},
          isFinished: true
        }}
        bind:grades={data.questionAnswerByPoint}
        bind:reasons
        bind:isGradeWithAI
        bind:isLoading
        disableGrading={false}
      />
    {/if}
  </div>
  <div class="ml-4 mt-2 w-2/5 sticky top-0">
    <div class="border border-gray-300 rounded-md">
      <div
        class="flex gap-1 justify-between items-center border-b border-t-0 border-l-0 border-r-0 border-gray-300 p-3"
      >
        <p class="dark:text-white font-bold text-base">
          {$t('course.navItem.submissions.grading_modal.details')}
          {#if data.isEarly}
            <span class="ml-2 text-sm badge rounded-sm px-2 bg-green-500 text-white">
              {$t('course.navItem.submissions.grading_modal.early')}</span
            >
          {:else}
            <span class="ml-2 badge text-sm rounded-sm px-2 bg-red-500 text-white">
              {$t('course.navItem.submissions.grading_modal.late')}
            </span>
          {/if}
        </p>

        <OverflowMenu open={openMenu} flipped>
          <OverflowMenuItem
            text={$t('delete_modal.label')}
            on:click={() => {
              openDeletePrompt = true;
              openMenu = false;
            }}
            danger
          />
        </OverflowMenu>
      </div>

      <div class="flex items-center space-x-4 text-sm px-3 py-2">
        <p class="dark:text-white text-sm text-gray-500 font-semibold">
          {$t('course.navItem.submissions.grading_modal.total_grade')}:
        </p>

        <Tag
          class="dark:text-white font-semibold text-black bg-gray-100 dark:bg-neutral-700 rounded-md w-fit"
        >
          {total}/{maxPoints}
        </Tag>
      </div>
      <!-- <div class="flex items-center text-sm p-3">
        <p class="dark:text-white font-bold w-1/2">Status</p>
        <div class="flex items-center">
          <p class="dark:text-white rounded-full w-5 h-5 bg-yellow-300 mr-2" />
          <p class="dark:text-white">Grading</p>
        </div>
      </div> -->
      <div class="flex items-center space-x-4 text-sm px-3 py-2">
        <p class="dark:text-white text-sm text-gray-500 font-semibold">
          {$t('course.navItem.submissions.grading_modal.student')}:
        </p>
        {#if data.student}
          <div
            class="flex flex-row justify-center items-center bg-gray-100 dark:bg-neutral-700 rounded-md p-[6px]"
          >
            <img
              alt="Student avatar"
              class="flex rounded-full h-5 w-5"
              src={data.student.avatar_url}
            />
            <p class="dark:text-white font-semibold ml-2 text-sm line-clamp-1">
              {data.student.fullname}
            </p>
          </div>
        {/if}
      </div>
      <!-- <div class="flex items-center space-x-4 text-sm px-3 py-2">
        <p class="dark:text-white text-sm text-gray-500 font-semibold">Assesment Type:</p>
        <Tag
          class="dark:text-white font-semibold bg-gray-100 dark:bg-neutral-700 rounded-md text-black w-fit"
          >Paragraph</Tag
        >
      </div> -->

      <div class="flex flex-col items-start text-sm px-3 py-2">
        <p class="dark:text-white text-gray-500 font-semibold">
          {$t('course.navItem.submissions.grading_modal.status')}:
        </p>
        <Dropdown
          bind:selectedId
          items={SELECTABLE_STATUS}
          class="w-full"
          on:select={handleStatusChange}
        />
      </div>

      <div class="flex flex-col items-start text-sm px-3 py-2">
        <p class="dark:text-white text-gray-500 font-semibold">
          {$t('course.navItem.submissions.grading_modal.add_comment')}:
        </p>
        <TextArea
          bgColor="bg-gray-100 dark:bg-neutral-700"
          className="font-semibold"
          placeholder={$t('course.navItem.submissions.grading_modal.add_comment_placeholder')}
          bind:value={data.feedback}
        />
      </div>

      <div class="flex flex-col w-full space-y-3 px-3 py-2">
        <PrimaryButton
          onClick={gradeWithAI}
          variant={VARIANTS.OUTLINED}
          className="space-x-3 py-3 px-8 w-full "
        >
          <img src="/ai.svg" alt="ai" />
          <p class="font-semibold text-sm">
            {$t('course.navItem.submissions.grading_modal.grade_with_ai')}
          </p>
        </PrimaryButton>
        <PrimaryButton
          onClick={() => {
            handleSave(data);
            // onClose();
          }}
          isLoading={isSaving}
          label={$t('course.navItem.submissions.grading_modal.submit_grades')}
          variant={VARIANTS.CONTAINED}
          className="py-3 px-8 w-full"
        />
      </div>
      <!-- <div class="flex items-center text-sm p-3">
        <p class="dark:text-white font-bold w-1/2">Teacher</p>
        <p class="dark:text-white">rotimi-best</p>
      </div> -->
    </div>
  </div>
</Modal>

<style>
  .badge {
    width: fit-content;
  }
</style>
