---
layout: post
title: PintOS process_wait 올바르게 구현하기
subtitle: 크래프톤 정글에서 PintOS 구현하기
tags: [OS, C]
author: Im Yongsik
comments: True
use_math: True
---

## PintOS에서 process_wait 올바르게 구현하기

### 0. 서론

2주차에서 구현한 코드 중 `process_wait` 및 `process_exit`가 심각한 오류를 내포하고 있음을 발제를 통해 확인할 수 있었다. 이를 해소하기 위한 방법을 알아보자.

### 1. 기존 코드의 문제점

기존 `process_wait()`과 `process_exit()`은 아래와 같이 구현되어 있었다.

{%highlight c%}

/* process_wait() */
int process_wait(tid_t child_tid UNUSED)
{
    /* XXX: Hint) The pintos exit if process_wait (initd), we recommend you
     * XXX:       to add infinite loop here before
     * XXX:       implementing the process_wait. */

    struct thread* curr = thread_current();
    struct thread* child = get_child(child_tid);
    if (child == NULL)
    {
        return -1;
    }
    sema_down(&child->sema_for_wait);
    int child_exit_status = child->exit_status;
    list_remove(&child->child_elem);
    sema_up(&child->sema_for_free);
    return child_exit_status;
}

/* process_exit() */
void process_exit(void)
{
    struct thread* cur = thread_current();
    /* TODO: Your code goes here.
     * TODO: Implement process termination message (see
     * TODO: project2/process_termination.html).
     * TODO: We recommend you to implement process resource cleanup here. */

    // 프로세스의 모든 열린 파일 close
    for (int fd = 0; fd < OPEN_MAX, fd++;)
    {
        close(fd);
    }
    
    /* Deallocate File Descriptor Table */
    palloc_free_multiple(cur->fdt, FDT_PAGES);
    file_close(cur->running_f);
    
    sema_up(&cur->sema_for_wait);
    sema_down(&cur->sema_for_free);
    
    process_cleanup();
}

{%endhighlight%}

여기서 문제가 되는 코드는 `process_exit()`의

{%highlight c%}

sema_down(&cur->sema_for_free);

{%endhighlight%}

이다.

위 코드에 의해 exit되는 process들은 전부 세마포의 값을 내리고 스스로 block상태가 되게 되는데, 만약 해당 프로세스의 부모 프로세스가 `wait()` 시스템 콜을 호출하지 않는다면 이들은 영원히 깨어나지 못하게 된다.

즉, `sema_down()`은 항상 호출되는데 `sema_up()`은 조건부로 호출되므로 문제가 발생한다.

따라서 이를 위해선 문제가 되는 `sema_down()`과 `sema_up()`을 호출하지 않아야 하므로 아래와 같은 코드가 되어야 한다.

{%highlight c%}

/* process_wait() */
int process_wait(tid_t child_tid UNUSED)
{
    /* XXX: Hint) The pintos exit if process_wait (initd), we recommend you
     * XXX:       to add infinite loop here before
          * XXX:       implementing the process_wait. */

    struct thread* curr = thread_current();
    struct thread* child = get_child(child_tid);
    if (child == NULL)
    {
        return -1;
    }
    sema_down(&child->sema_for_wait);
    int child_exit_status = child->exit_status;
    list_remove(&child->child_elem);
    return child_exit_status;
}

/* process_exit() */
void process_exit(void)
{
    struct thread* cur = thread_current();
    /* TODO: Your code goes here.

     * TODO: Implement process termination message (see
          * TODO: project2/process_termination.html).
          * TODO: We recommend you to implement process resource cleanup here. */

    // 프로세스의 모든 열린 파일 close
    for (int fd = 0; fd < OPEN_MAX, fd++;)
    {
        close(fd);
    }
    
    /* Deallocate File Descriptor Table */
    palloc_free_multiple(cur->fdt, FDT_PAGES);
    file_close(cur->running_f);
    
    sema_up(&cur->sema_for_wait);
    
    process_cleanup();
}

{%endhighlight%}

그런데 이렇게 두 번째 세마포를 사용하지 않으면 발생하는 또 다른 문제가 있다(사실 이 문제 때문에 두 번째 세마포`sema_for_free`를 사용했다).

{%highlight c%}

/* process_wait() */
{
	//...
    sema_down(&child->sema_for_wait);
    int child_exit_status = child->exit_status;
    list_remove(&child->child_elem); // Warning!
    return child_exit_status;
}

{%endhighlight%}

바로 child thread가 `process_exit()`이후 제거되게 되면 `list_remove(&child->child_elem)`의 child 포인터가 댕글링 포인터가 된다는 점이다.

### 2. 해결 방안

따라서 이를 해소하기 위해서 댕글링 포인터가 발생할 가능성을 완전히 배제하기 위해 부모가 child pointer를 관리하지 않도록 수정했고, 대신 child의 tid와 exit_status를 리스트 형태로 가지도록 하였다.

{%highlight c%}

struct child_info_t
{
    tid_t tid;
    int exit_status;
    struct semaphore sema_for_wait;
    struct list_elem elem;
};

{%endhighlight%}

또, `sync_read`나 `sync_write`등의 테스트를 통과하기 위해 각 자식 정보마다 고유의 `sema_for_wait`을 가지도록 했다.

그리고 해당 구조체를 위한 인터페이스들을 정의해 주었다.

{%highlight c%}

/////////////// declaration ///////////////
void add_child(struct thread* parent, tid_t child_tid);
void set_exit_status(struct thread* parent, tid_t child_tid, int exit_status);
int get_exit_status(struct thread* parent, tid_t child_tid);
struct child_info_t* get_child_info(struct thread* parent, tid_t child_tid);
void delete_child(struct thread* parent, tid_t child_tid);

/////////////// definition ///////////////
void add_child(struct thread* parent, tid_t child_tid)
{
    struct child_info_t* info = (struct child_info_t*)malloc(sizeof(struct child_info_t));
    info->tid = child_tid;
    sema_init(&info->sema_for_wait, 0);
    list_push_back(&parent->child_info_list, &info->elem);
}

void set_exit_status(struct thread* parent, tid_t child_tid, int exit_status)
{
    struct list_elem* e;
    for (e = list_begin(&parent->child_info_list); e != list_end(&parent->child_info_list); e = list_next(e))
    {
        struct child_info_t* info = list_entry(e, struct child_info_t, elem);
        if (info->tid == child_tid)
        {
            info->exit_status = exit_status;
        }
    }
}

int get_exit_status(struct thread* parent, tid_t child_tid)
{
    struct list_elem* e;
    for (e = list_begin(&parent->child_info_list); e != list_end(&parent->child_info_list); e = list_next(e))
    {
        struct child_info_t* info = list_entry(e, struct child_info_t, elem);
        if (info->tid == child_tid)
        {
            return info->exit_status;
        }
    }

    return 0;
}

struct child_info_t* get_child_info(struct thread* parent, tid_t child_tid)
{
    struct list_elem* e;
    for (e = list_begin(&parent->child_info_list); e != list_end(&parent->child_info_list); e = list_next(e))
    {
        struct child_info_t* info = list_entry(e, struct child_info_t, elem);
        if (info->tid == child_tid)
        {
            return info;
        }
    }

    return NULL;
}

void delete_child(struct thread* parent, tid_t child_tid)
{
    struct list_elem* e;
    for (e = list_begin(&parent->child_info_list); e != list_end(&parent->child_info_list); e = list_next(e))
    {
        struct child_info_t* info = list_entry(e, struct child_info_t, elem);
        if (info->tid == child_tid)
        {
            list_remove(e);
        }
    }
}

{%endhighlight%}

이후 `process_wait()` 과 `process_exit()`을 아래와 같이 수정해준다.

{%highlight c%}

int process_wait(tid_t child_tid UNUSED)
{
    struct thread* curr = thread_current();
    struct child_info_t* info = NULL;
    if (!(info = get_child_info(curr, child_tid)))
    {
        return -1;
    }
    sema_down(&info->sema_for_wait);
    int child_exit_status = get_exit_status(curr, child_tid);
    delete_child(curr, child_tid);
    return child_exit_status;
}

void process_exit(void)
{
    struct thread* cur = thread_current();

    // 프로세스의 모든 열린 파일 close
    for (int fd = 0; fd < OPEN_MAX, fd++;)
    {
        close(fd);
    }
    
    /* Deallocate File Descriptor Table */
    palloc_free_multiple(cur->fdt, FDT_PAGES);
    file_close(cur->running_f);
    struct child_info_t* info = get_child_info(cur->parent, cur->tid);
    sema_up(&info->sema_for_wait);
    
    process_cleanup();
}

{%endhighlight%}

이외에도 `syscall.c`등에서 시스템 콜 함수들을 조금씩 바꾸어야 하는데, 그 부분은 간단하므로 생략한다.
